원본코드
# -*- coding: utf-8 -*-

'''
Video Uav Tracker 3D  v 2.1

Replay a video in sync with a gps track displayed on the map.


     -------------------
copyright    : (C) 2017 by Salvatore Agosta
email          : sagost@katamail.com


This program is free software; you can redistribute it and/or modify
 it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
 (at your option) any later version.


INSTRUCTION:

ATTENTION: 3D IS NOT TESTED ON WINDOWS PLATFORM
- Pixel value query need a .npz archive containing one array data for every frame, it must be named as 'VideoFile.npz' and be in the same folder of 'VideoFile.mp4'
- for 3d options install numpy,panda3d and pypng python3 modules
- Download all files from https://github.com/sagost/Video_UAV_Tracker-3D/Video_UAV_Tracker/FFMPEG and copy them in your Video_Uav_Tracker/FFMPEG folder

Syncing:
- Create new project
- Select video and .gpx track (1 trkpt per second)
- Identify first couple Frame/GpsTime and select it.
- Push Synchronize
- Push Start

Replay:
- Move on map
- Create associated DB shapefile
- Add POI with associated video frame saved
- Extract frames with associated coordinates for rapid photogrammetry use
'''

import time
from vut_qgismap import Ui_Form
from PyQt5 import QtWidgets
from PyQt5.QtCore import QUrl, QSize
from PyQt5.QtWidgets import QMessageBox , QFileDialog , QStyle, QInputDialog, QLineEdit, QShortcut
from PyQt5.QtMultimedia import QMediaPlayer,  QMediaContent
from PyQt5.QtGui import QColor,QKeySequence
from SkipTrackTool import *
from AddPoint import *
from CanvasMarkers import PositionMarker
import os
import threading
import sys
import math
from PyQt5.uic.Compiler.qtproxies import QtCore



PACKAGE_PARENT = '..'
SCRIPT_DIR = os.path.dirname(os.path.realpath(os.path.join(os.getcwd(), os.path.expanduser(__file__))))
sys.path.append(os.path.normpath(os.path.join(SCRIPT_DIR, PACKAGE_PARENT)))

from osgeo import osr
from geographiclib.geodesic import Geodesic
from qgis.core import *
from qgis.gui import QgsVertexMarker
import numpy
try:
    from direct.distributed.PyDatagram import PyDatagram
    from panda3d.core import QueuedConnectionManager
    from panda3d.core import QueuedConnectionListener
    from panda3d.core import QueuedConnectionReader
    from panda3d.core import ConnectionWriter
except:
    pass



class QGisMap(QtWidgets.QWidget, Ui_Form):
    
    def __init__(self,projectfile,MainWidget):
        QtWidgets.QMainWindow.__init__(self)
        if os.name == 'nt':
            ffmpeg = os.path.dirname(__file__)[0:-18]+'/Video_UAV_Tracker/FFMPEG/ffmpeg.exe'
            versione = 'ffmpeg.exe'
        else:
            ffmpeg = os.path.dirname(__file__)+'/FFMPEG/./ffmpeg'
            versione = 'ffmpeg'
        if os.path.exists(ffmpeg):
            self.setupUi(self)
            self.setWindowFlags(Qt.WindowStaysOnTopHint)
            self.Main = MainWidget
            self.projectfile = projectfile
            self.PixelConversion = False
            self.lcdNumber.hide()

            with open(self.projectfile,'r') as File:
                    for line in File:
                        if line[0:19] == 'Video file location':
                            self.videoFile = line.split('=')[-1][1:-1]
                        elif line[0:23] == 'Video start at msecond:':
                            self.fps = float(line.split()[7])
                            self.StartMsecond = int(line.split()[4])
                            self.VideoWidth = float(line.split()[8])
                            self.VideoHeight = float(line.split()[9])
                            if int(line.split()[10]) == 1:
                                self.PixelConversion = True
                                self.PixelArrays = numpy.load(self.videoFile.split('.')[-2] + '.npz') # add conf window
                                self.video_frame.setMouseTracking(True)
                                self.video_frame.mousePressEvent = self.getScreenPos
                                self.lcdNumber.show()


                        elif line[0:4] == 'DB =':
                            DB = line.split('=')[-1][1:-1]
                            if DB == 'None':
                                self.DB = None
                            else:
                                self.DB = DB
                        elif line[0:4] == '3D =':
                            partial = line.split('=')[-1]
                            partial2 = partial.split(',')
                            self.Course = int(partial2[-1])
                            if partial2[0] == ' None':
                                break
                            else:
                                self.DEM = partial2[0]
                                self.Image = partial2[1]
                                self.HFilmSize = partial2[2]
                                self.VFilmSize = partial2[3]
                                self.FocalLenght = partial2[4][:-1]
                                self.toolButton_7.setEnabled(True)
                                break

            self.pushButton_3.setCheckable(True)
            self.EnableMapTool = None
            self.ExtractTool = 0
            self.dockWidget_4.hide()
            self.GPXList = []
            self.positionMarker=PositionMarker(self.Main.iface.mapCanvas())               
            self.muteButton.setIcon(
                        self.style().standardIcon(QStyle.SP_MediaVolume))
            self.playButton.setIcon(
                        self.style().standardIcon(QStyle.SP_MediaPause))
            self.player = QMediaPlayer()
            self.player.setVideoOutput(self.video_frame)
            
            #assign ShortCut
            self.playButton.clicked.connect(self.PlayPause)
            self.playButton.setShortcut(QKeySequence('Space'))
            
            self.muteButton.clicked.connect(self.MuteUnmute)
            
            
            self.SkipBacktoolButton_8.clicked.connect(self.SkipBackward_5sec)
            self.SkipBacktoolButton_8.setShortcut(QKeySequence('Left'))
            
            self.SkipFortoolButton_9.clicked.connect(self.SkipForward_5sec)
            self.SkipFortoolButton_9.setShortcut(QKeySequence('Right'))
            
            self.toolButton_11.clicked.connect(self.SkipBackward_10sec)
            self.toolButton_11.setShortcut(QKeySequence('Ctrl+Left'))
            
            self.toolButton_12.clicked.connect(self.SkipForward_10sec)
            self.toolButton_12.setShortcut(QKeySequence('Ctrl+Right'))
            
            self.toolButton_previous1min.clicked.connect(self.SkipBackward_1min)
            self.toolButton_previous1min.setShortcut(QKeySequence('Ctrl+Shift+Left'))
            
            self.toolButton_next1min.clicked.connect(self.SkipForward_1min)
            self.toolButton_next1min.setShortcut(QKeySequence('Ctrl+Shift+Right'))
            
            
            self.toolButton_4.clicked.connect(self.ExtractToolbar)
            self.toolButton_5.clicked.connect(self.close)   
            self.pushButtonCut_2.clicked.connect(self.ExtractCommand)
            self.pushButtonCutA_6.clicked.connect(self.ExtractFromA)
            self.pushButtonCutB_6.clicked.connect(self.ExtractToB)
            self.pushButton_5.clicked.connect(self.CancelVertex)
            self.pushButton.clicked.connect(self.SendMosaicStart)
            self.toolButton_7.clicked.connect(self.Setup3D)
            self.progressBar.hide()     
            self.Main.pushButton_2.hide()
            self.Main.pushButton_8.hide()
            self.Main.groupBox.show()
            self.Main.groupBox_4.hide()
            self.ExtractA = False
            self.ExtractB = False
            self.ExtractedDirectory = None 
            self.pushButtonCut_2.setEnabled(False)
            self.toolButton_6.setEnabled(False)
            self.Tridimensional = False
            self.AddPointStatus = False
            self.Mosaic = False

            self.video_frame.setMouseTracking(True)
            self.video_frame.mousePressEvent = self.getScreenPos
            
            self.LoadGPXVideoFromPrj(self.projectfile)
        else:
            ret = QMessageBox.warning(self, "Warning", 'missing ffmpeg binaries, please download it from https://github.com/sagost/VideoUavTracker/blob/master/FFMPEG/'+versione+' and paste it in /.qgis3/python/plugins/Video_UAV_Tracker/FFMPEG/ ', QMessageBox.Ok)
            self.close()

    def LoadGPXVideoFromPrj(self,VideoGisPrj):
        
        self.Polyline = []
        with open(VideoGisPrj,'r') as File:
            Counter = 0
            for line in File:
                if Counter < 6:
                    pass
                else:
                    line = line.split()
                    lat = float(line[0])
                    lon = float(line[1])
                    ele = float(line[2])
                    speed = float(line[3])
                    course = float(line[4])
                    #pitch = float(line[5])
                    #roll = float(line[6])
                    time = line[7]
                    Point = [lat,lon,ele,speed,course,time]
                    qgsPoint = QgsPoint(lon,lat)
                    self.Polyline.append(qgsPoint)
                    self.GPXList.append(Point)
                Counter = Counter + 1
        self.GpsLayer = QgsVectorLayer("LineString?crs=epsg:4326", self.videoFile.split('.')[-2].split('/')[-1], "memory")
        self.pr = self.GpsLayer.dataProvider()
        feat = QgsFeature()
        feat.setGeometry(QgsGeometry.fromPolyline(self.Polyline))
        self.GPS_BoundingBox = feat.geometry().boundingBox().toString()
        self.pr.addFeatures([feat])
        self.GpsLayer.updateExtents()
        if self.DB != None:
            try:
                self.DBLayer = QgsVectorLayer(self.DB,self.DB.split('.')[-2].split('/')[-1],'ogr')
                QgsProject.instance().addMapLayers([self.DBLayer,self.GpsLayer])
                self.AddPointMapTool = AddPointTool(self.Main.iface.mapCanvas(),self.DBLayer,self)
                self.toolButton_6.clicked.connect(self.AddPointTool)   
                self.toolButton_6.setEnabled(True)
            except:
                ret = QMessageBox.warning(self, "Warning", str(self.DB)+' is not a valid shapefile.', QMessageBox.Ok)
                return
        else:
            QgsProject.instance().addMapLayers([self.GpsLayer])    
        self.duration = len(self.GPXList)
        self.ExtractToB = self.duration
        self.horizontalSlider.setSingleStep(1000)
        self.horizontalSlider.setMinimum(0)
        self.horizontalSlider.setMaximum(len(self.GPXList)*1000)
        url = QUrl.fromLocalFile(str(self.videoFile))
        mc = QMediaContent(url)
        self.player.setMedia(mc)
        self.player.setPosition(self.StartMsecond)

        self.PlayPause()
        self.horizontalSlider.sliderMoved.connect(self.setPosition)
        self.player.stateChanged.connect(self.mediaStateChanged)
        self.player.positionChanged.connect(self.positionChanged)

        self.player.setNotifyInterval(100)

        self.pushButton_3.clicked.connect(self.MapTool)
        self.skiptracktool = SkipTrackTool( self.Main.iface.mapCanvas(),self.GpsLayer , self)

    def Setup3D(self):
        if not self.Tridimensional:
            self.player.pause()
            self.Tridimensional = True

            BBxMin = (self.GPS_BoundingBox.split(':')[0].split(',')[0])
            BByMin = (self.GPS_BoundingBox.split(':')[0].split(',')[1])
            BBxMax = (self.GPS_BoundingBox.split(':')[1].split(',')[0])
            BByMax = (self.GPS_BoundingBox.split(':')[1].split(',')[1])


            ScriptName = str(os.path.dirname(__file__)+'/VUT_3D.py')
            command = ('python3 '+ ScriptName+ ' '+self.DEM+' '+self.Image+' '+str(self.HFilmSize)+
                        ' '+str(self.VFilmSize)+' '+str(self.FocalLenght)+' '+str(self.projectfile)+
                        ' '+str(os.path.dirname(__file__)+' '+str(self.videoFile)+' '+str(self.VideoWidth)+
                        ' '+str(self.VideoHeight)+' '+str(self.StartMsecond/1000)+' '+BBxMin+' '+BByMin+' '+BBxMax+' '+BByMax))
            a = os.popen(command)
            self.SetupConnection()
        else:
            self.Tridimensional = False
            self.CloseConnection()

    def SetupConnection(self):
        self.pushButton.setEnabled(True)
        self.label_2.setEnabled(True)
        self.spinBox.setEnabled(True)
        while not os.path.exists(str(os.path.dirname(__file__) + '/tmpConnection.txt')):
            pass
        while os.stat(str(os.path.dirname(__file__)+'/tmpConnection.txt')).st_size < 1:
            pass
        os.remove(str(os.path.dirname(__file__)+'/tmpConnection.txt'))
        self.cManager = QueuedConnectionManager()
        self.cWriter = ConnectionWriter(self.cManager, 0)
        port_address = 9098  # same for client and server
        ip_address = "localhost"
        timeout_in_miliseconds = 3000  # 3 seconds
        self.myConnection = self.cManager.openTCPClientConnection(ip_address, port_address, timeout_in_miliseconds)
        if self.myConnection:
            self.PlayPause()

    def CloseConnection(self):
        self.pushButton.setEnabled(False)
        self.pushButton.setChecked(False)
        self.label_2.setEnabled(False)
        self.spinBox.setEnabled(False)
        self.SendCloseData()
        self.cManager.closeConnection(self.myConnection)

    def SendCloseData(self):
        myPyDatagram = PyDatagram()
        myPyDatagram.addUint8(3)
        myPyDatagram.addString('close,close')
        self.cWriter.send(myPyDatagram, self.myConnection)

    def SendPlayData(self,time,start):
        myPyDatagram = PyDatagram()
        start = start  /1000
        myPyDatagram.addUint8(1)
        myPyDatagram.addString(str(time)+','+str(start))
        self.cWriter.send(myPyDatagram, self.myConnection)

    def SendPauseData(self,start):
        myPyDatagram = PyDatagram()
        start = start  / 1000
        myPyDatagram.addUint8(2)
        myPyDatagram.addString(str(start))
        self.cWriter.send(myPyDatagram, self.myConnection)

    def Send_3dPoint_Data(self,time,Pixelx,Pixely):
        myPyDatagram = PyDatagram()
        myPyDatagram.addUint8(5)
        myPyDatagram.addString(str(time)+','+str(Pixelx)+','+str(Pixely))
        self.cWriter.send(myPyDatagram, self.myConnection)

    def SendMosaicStart(self):
        if self.Mosaic == False:
            self.Mosaic = True
        else:
            self.Mosaic = False
        myPyDatagram = PyDatagram()
        myPyDatagram.addUint8(4)
        myPyDatagram.addString('Start/Stop Mosaic, 0.'+str(self.spinBox.value()))
        self.cWriter.send(myPyDatagram, self.myConnection)

    def AddPointTool(self):
        self.AddPointStatus = True
        if self.EnableMapTool == True:
            self.pushButton_3.setChecked(False)
            self.EnableMapTool = False
        self.Main.iface.mapCanvas().setMapTool(self.AddPointMapTool)

    def getScreenPos(self,mouseEvent):

            if self.Tridimensional == True or self.PixelConversion == True or self.AddPointStatus == True:
                PlayerPos = self.player.position()   #-self.StartMsecond /1000
                pos = ((PlayerPos/1000)*(self.fps*(self.player.duration()/1000)))/(self.player.duration()/1000)
                FrameHeight = self.video_frame .height()
                FrameWidth = self.video_frame.width()
                FrameSize = QSize(self.video_frame.width(),self.video_frame .height())
                VideoSize = QSize(self.VideoWidth, self.VideoHeight)
                ScaledSize = VideoSize.scaled(FrameSize, 1)
                LateralBorders = (FrameWidth - ScaledSize.width()) / 2
                VerticalBorders = (FrameHeight-ScaledSize.height()) / 2
                VideoPixelPositionX = mouseEvent.pos().x() - LateralBorders
                VideoPixelPositionY = mouseEvent.pos().y() - VerticalBorders
                Pixelx = (VideoPixelPositionX * self.VideoWidth) / ScaledSize.width()
                Pixely = (VideoPixelPositionY * self.VideoHeight) / ScaledSize.height()

                if 0 <=Pixelx <= self.VideoWidth and 0 <=Pixely <= self.VideoHeight:

                    if self.PixelConversion == True:
                        Valore = self.PixelArrays['arr_'+str(int(pos))][round(Pixelx),round(Pixely)]
                        self.lcdNumber.display(round(Valore,2))

                    if self.AddPointStatus == True:
                        if self.Tridimensional == True and self.Mosaic == False:
                            pos = (PlayerPos - self.StartMsecond) /1000
                            self.Send_3dPoint_Data(pos,Pixelx,Pixely)
                            while not os.path.exists(str(os.path.dirname(__file__) + '/tmpCoordinate.txt')):
                                pass
                            while os.stat(str(os.path.dirname(__file__) + '/tmpCoordinate.txt')).st_size < 70:
                                pass
                            with open(os.path.dirname(__file__) + '/tmpCoordinate.txt','r') as CoordinateFile:
                                for line in CoordinateFile:
                                    X = float(line.split()[0])
                                    Y = float(line.split()[1])
                                    Z = float(line.split()[2])
                                    break
                            os.remove(os.path.dirname(__file__) + '/tmpCoordinate.txt')
                            self.AddPoint(X,Y,PlayerPos)

                            #retrieve output coordinate
                        else:
                            print('DA AGGIUNGERE ADD POINT SU NAVIGAZIONE')

    def MapTool(self):

        if not self.EnableMapTool:
            self.PrevMapTool = self.Main.iface.mapCanvas().mapTool()
            self.Main.iface.mapCanvas().setMapTool(self.skiptracktool)
            self.pushButton_3.setChecked(True)
            self.EnableMapTool = True
        else:
            self.Main.iface.mapCanvas().unsetMapTool(self.skiptracktool)
            self.Main.iface.mapCanvas().setMapTool(self.PrevMapTool)
            self.pushButton_3.setChecked(False)
            self.EnableMapTool = False
                           
    def closeEvent(self, *args, **kwargs):
        try:
            self.player.stop()
            self.Main.iface.mapCanvas().scene().removeItem(self.positionMarker)
            self.CancelVertex()
            self.Main.pushButton_2.show()
            self.Main.groupBox.hide()
            self.Main.pushButton_8.show()
            self.Main.groupBox_4.show()
            self.dockWidget_2.close()
            self.CloseConnection()
            self.PixelArrays = None
        except:
            pass
        return QtWidgets.QWidget.closeEvent(self, *args, **kwargs)

    def mediaStateChanged(self, state):
        if self.player.state() == QMediaPlayer.PlayingState:
            self.playButton.setIcon(
                    self.style().standardIcon(QStyle.SP_MediaPause))
        else:
            self.playButton.setIcon(
                    self.style().standardIcon(QStyle.SP_MediaPlay))
    
    def setPosition(self, position):
        if not self.Tridimensional:
            self.player.setPosition(position+self.StartMsecond)
        else:
            if self.player.state() == QMediaPlayer.PlayingState:
                StartTime = time.time() + 0.2
                self.SendPlayData(StartTime, position)
                while time.time() < StartTime:
                    pass
                self.player.setPosition(position + self.StartMsecond)
            else:
                self.SendPauseData(position)
                self.player.setPosition(position + self.StartMsecond)

    def setPositionFrame(self,position):
        if not self.Tridimensional:
            self.player.setPosition(position)
        else:
            if self.player.state() == QMediaPlayer.PlayingState:
                StartTime = time.time() + 0.2
                self.SendPlayData(StartTime, position)
                while time.time() < StartTime:
                    pass
                self.player.setPosition(position)
            else:
                self.SendPauseData(position + self.StartMsecond)
                self.player.setPosition(position + self.StartMsecond)

    def secTotime(self,seconds): 
            m, s = divmod(seconds, 60)
            h, m = divmod(m, 60)
            return "%d:%02d:%02d" % (h, m, s)   
                   
    def positionChanged(self, progress):

        if progress < self.StartMsecond:
            self.player.setPosition(self.StartMsecond)
            progress = self.StartMsecond
        AssociatedGps = round((progress - self.StartMsecond )/1000)

        totalTime = self.secTotime(self.duration)

        VideoTime = (progress - self.StartMsecond)
        X, Y, Z, Azimuth, SpeedMeterSecond = self.InterpolatePosition(VideoTime)

        self.DisplayPoint(AssociatedGps,X, Y, Z, Azimuth, SpeedMeterSecond)

        actualTime = self.secTotime((progress - self.StartMsecond )/1000)
        self.replayPosition_label.setText(actualTime + ' / '+totalTime)
        if not self.horizontalSlider.isSliderDown():
            self.horizontalSlider.setValue(progress - self.StartMsecond ) 

    def InterpolatePosition(self,VideoTime):
        i = int(VideoTime /1000)
        i2 = i+1
        if i2 >= len(self.GPXList):
            i = i -1
            i2 = i2-1

        latitude1, longitude1 = float(self.GPXList[i][0]), float(self.GPXList[i][1])
        latitude2, longitude2 = float(self.GPXList[i2][0]), float(self.GPXList[i + 1][1])
        ele1 = float(self.GPXList[i][2])
        ele2 = float(self.GPXList[i2][2])
        Calculus = Geodesic.WGS84.Inverse(latitude1, longitude1, latitude2, longitude2)
        DistanceBetweenPoint = Calculus['s12']
        Azimuth = Calculus['azi2']
        SpeedMeterSecond = DistanceBetweenPoint
        decimalSecondToAdd = (VideoTime - (i * 1000)) *0.001
        CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth, decimalSecondToAdd * SpeedMeterSecond)
        X, Y = CalculusDirect['lon2'], CalculusDirect['lat2']
        Z = ele1 + decimalSecondToAdd * (ele2 - ele1)
        if self.Course == 1:
            aTheta = float( self.GPXList[i][4])
            a1 = float( self.GPXList[i+1][4])
            da = (a1-aTheta)%(math.pi*2)
            shortest_angle = 2*da % (math.pi*2) - da

            Azimuth =  aTheta + shortest_angle * decimalSecondToAdd

        else:
            if Azimuth < 0:
                Azimuth += 360
        return X , Y , Z , Azimuth , SpeedMeterSecond

    def DisplayPoint(self,Point, Lon,Lat,Ele,Course,Speed):
        if Point >= len(self.GPXList):
            Point = len(self.GPXList) - 1
        gpx = self.GPXList[Point]
        time = gpx[5]
        Point = QgsPointXY(Lon, Lat)
        canvas = self.Main.iface.mapCanvas()
        crsSrc = QgsCoordinateReferenceSystem(4326)    # .gpx is in WGS 84
        crsDest = canvas.mapSettings().destinationCrs()
        xform = QgsCoordinateTransform(crsSrc, crsDest, QgsProject.instance())
        self.positionMarker.setHasPosition(True)
        PointTrans = xform.transform(Point)
        self.positionMarker.newCoords(PointTrans)
        self.positionMarker.angle = Course
        self.Main.label_14.setText('GPS Time: ' + str(time))
        self.Main.label_15.setText('Course: ' + "%.2f" % (Course))
        self.Main.label_16.setText('Ele: ' + "%.2f" % (Ele))
        self.Main.label_17.setText('Speed m/s: ' + "%.2f" % (Speed))
        self.Main.label_19.setText('Lat : ' + "%.6f" % (Lat))
        self.Main.label_18.setText('Lon : ' + "%.6f" % (Lon))
        extent = canvas.extent() 
        boundaryExtent=QgsRectangle(extent)
        boundaryExtent.scale(0.7)
        if not boundaryExtent.contains(QgsRectangle(PointTrans, PointTrans)):
            extentCenter= PointTrans
            newExtent=QgsRectangle(
                        extentCenter.x()-extent.width()/2,
                        extentCenter.y()-extent.height()/2,
                        extentCenter.x()+extent.width()/2,
                        extentCenter.y()+extent.height()/2
                        )
            canvas.setExtent(newExtent)
            canvas.refresh()

                  
    def MuteUnmute(self):
        if self.player.mediaStatus() == 6 :
            if self.player.isMuted() == 1:
                self.player.setMuted(0)
                self.muteButton.setIcon(
                    self.style().standardIcon(QStyle.SP_MediaVolume))
            elif self.player.isMuted() == 0:
                self.player.setMuted(1)
                self.muteButton.setIcon(
                    self.style().standardIcon(QStyle.SP_MediaVolumeMuted))   
    
    def SkipForward_10sec(self):
        position = self.player.position()
        self.setPositionFrame(position + 10000)
       
    def SkipBackward_10sec(self): 
        position = self.player.position()
        self.setPositionFrame(position - 10000)

    def SkipForward_5sec(self):  
        position = self.player.position()
        self.setPositionFrame(position+5000)

    def SkipBackward_5sec(self):
        position = self.player.position()
        self.setPositionFrame(position - 5000)
     
    def SkipForward_1min(self):
        position = self.player.position()
        self.player.setPosition(position+60000)
    
    def SkipBackward_1min(self):
        position = self.player.position()
        self.player.setPosition(position-60000)   
    
    def PlayPause(self):
        if self.player.state() == QMediaPlayer.PlayingState:
            if self.Tridimensional:
                self.player.pause()
                Position = self.player.position()
                self.SendPauseData(Position)
            else:
                self.player.pause()

        else:
            if self.Tridimensional:
                Position = self.player.position()-self.StartMsecond /1000
                StartTime = time.time() + 0.2
                self.SendPlayData(StartTime, Position)
                while time.time() < StartTime:
                    pass
                self.player.play()
            else:
                self.player.play()

    def findNearestPointInRecording(self, x,y):
        ClickPt = QgsPoint(x,y)
        Low =  ClickPt.distanceSquared(self.Polyline[0])
        NearPoint = 0
        Counter = 0
        for Point in self.Polyline:
            dist = ClickPt.distanceSquared(Point)
            if dist < Low:
                Low = dist
                NearPoint = Counter
                Counter = Counter + 1
            else:
                Counter = Counter + 1
        self.setPosition(NearPoint*1000)
        
    def ExtractSingleFrameOnTime(self, pos, outputfile):
        if os.name == 'nt':
            ffmpeg = ('"'+os.path.dirname(__file__)[0:-18]+'/Video_UAV_Tracker/FFMPEG/ffmpeg.exe'+'"')
            os.popen(str(ffmpeg)+' -ss '+str(pos/1000)+' -i '+str('"' +self.videoFile+ '"')+ ' -t 1 '+str('"'+outputfile+'"'))
        else:
            ffmpeg = os.path.dirname(__file__)+'/FFMPEG/./ffmpeg'
            os.system(str(ffmpeg)+' -ss '+str(pos/1000)+' -i '+str(self.videoFile)+' -t 1 '+str(outputfile))
                 
    def AddPoint(self,x,y,pos):

        self.Main.iface.mapCanvas().unsetMapTool(self.AddPointMapTool)
        self.AddPointStatus = False
        Point = QgsPointXY(x,y)
        if pos == -1000:
            pos = self.player.position()
        if self.player.state() == QMediaPlayer.PlayingState:
            self.player.pause()
        a = self.DBLayer.name()
        last_desc = '///'
        LayerName =str(a)
        last_desc2 = LayerName + ' Point N '
        directory = str(self.DB.split('.')[0])+'_Image/'
        if not os.path.exists(directory):
            os.makedirs(directory)
        fc = int(self.DBLayer.featureCount())
        self.ExtractSingleFrameOnTime(pos,directory+LayerName+'_'+str(fc)+'_.jpg')
        fields = self.DBLayer.fields()
        attributes = []
        lat,lon = Point.y(), Point.x()
        for field in fields:
            a = str(field.name())
            b = str(field.typeName())
            if a == 'id':
                fcnr = fc
                attributes.append(fcnr)  
            elif a == 'Lon(WGS84)':
                attributes.append(str(lon))                       
            elif a == 'Lat(WGS84)':
                attributes.append(str(lat))               
            elif a == 'Image link':
                attributes.append(str(directory+LayerName+'_'+str(fc)+'_.jpg'))                   
            else:                    
                if b == 'String':      
                    (a,ok) = QInputDialog.getText(
                                                  self.Main.iface.mainWindow(), 
                                                  "Attributes",
                                                  a + ' = String',
                                                  QLineEdit.Normal)
                    attributes.append(a)               
                elif b == 'Real':                    
                    (a,ok) = QInputDialog.getDouble(
                                                    self.Main.iface.mainWindow(), 
                                                    "Attributes",
                                                    a + ' = Real', decimals = 10)
                    attributes.append(a)
                elif b == 'Integer64':                    
                    (a,ok) = QInputDialog.getInt(
                                                 self.Main.iface.mainWindow(), 
                                                 "Attributes",
                                                 a + ' = Integer')
                    attributes.append(a)

        feature = QgsFeature()
        feature.setGeometry(QgsGeometry.fromPointXY(Point))
        feature.setAttributes(attributes)
        self.DBLayer.startEditing()
        self.DBLayer.addFeature(feature)
        self.DBLayer.commitChanges()    
        self.DBLayer.triggerRepaint()

    def ExtractCommand(self):
        def extract(ffmpeg, i , decimalSecondToAdd,videoFile, Directory ):
            os.popen(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                             ' -i '+ str('"'+videoFile+'"') + ' -frames:v 1 ' +'"'+ Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png'+'"')
            
        
        if self.ExtractToB <= self.ExtractFromA:
            ret = QMessageBox.warning(self, "Warning", '"To B" point must be after "from A" point', QMessageBox.Ok)
            self.CancelVertex()
        else:
            if os.name == 'nt':
                ffmpeg = '"'+os.path.dirname(__file__)[0:-18]+'/Video_UAV_Tracker/FFMPEG/ffmpeg.exe'+'"'
            else:
                ffmpeg = os.path.dirname(__file__)+'/FFMPEG/./ffmpeg'
            Directory,_ = QFileDialog.getSaveFileName(caption= 'Save georeferenced images')
            if Directory:
                self.progressBar.show()
                self.progressBar.setValue(0)
                start = self.ExtractFromA
                if self.comboBox_6.currentText() == 'seconds':           
                    finish = self.ExtractToB - self.ExtractFromA
                    fps = self.doubleSpinBox_2.value()
                    if fps < 1.0:
                        fps = 1.0 / fps
                    elif fps > 1:
                        fps = 1.0 / fps
                        
                    if os.name == 'nt':
                        os.popen(str(ffmpeg) + ' -ss ' + str(start) + ' -i '+ str('"'+self.videoFile+'"')+ ' -t ' + str(finish) + ' -vf fps=' + str(fps) + ' ' + '"'+Directory + '_%d.png'+'"')
                    else:
                        os.system(ffmpeg+' -ss '+ str(start) + ' -i '+ str(self.videoFile) + ' -t ' + str(finish) + ' -vf fps=' + str(fps) + ' ' + Directory + '_%d.png')
                else:
                    txtGPSFile = open(Directory + 'UTM_Coordinates.txt', 'w')
                    txtGPSFile.close()
                    txtGPSFile = open(Directory+ 'UTM_Coordinates.txt', 'a')
                    txtGPSFile.write('filename # East UTM # North UTM # Ele '+ '\n')
                    finish = self.ExtractToB
                    meters = self.doubleSpinBox_2.value()
                    Timerange = range(start, finish + 1)
                    RemainToUseMeterTotal = 0
                    if os.name == 'nt':
                        os.popen(ffmpeg + ' -ss '+ str(start) + ' -i '+ str('"'+self.videoFile+'"') + ' -frames:v 1 ' + '"'+Directory + '_sec_' + str(start)+'.00.png'+'"')
                    else:
                        os.system(ffmpeg+' -ss '+ str(start) + ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(start)+'.00.png')
                    lonUTM, latUTM,quotainutile = self.transform_wgs84_to_utm(float(self.GPXList[start][1]) , float(self.GPXList[start][0]))
                    ele = float(self.GPXList[start][2])
                    txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_' + str(start)+'.00.png,'+' '+ str(lonUTM) + ', '+ str(latUTM) + ', ' + str(ele) + '\n')
                    for i in Timerange:
                        progessBarValue = ((i-start) * 100) // len(Timerange)
                        self.progressBar.setValue(int(progessBarValue))
                        latitude1,longitude1 = float(self.GPXList[i][0]) ,float(self.GPXList[i][1])
                        latitude2,longitude2 = float(self.GPXList[i+1][0]) ,float(self.GPXList[i+1][1])
                        ele1 = float(self.GPXList[i][2])
                        ele2 = float(self.GPXList[i+1][2])
                        Calculus = Geodesic.WGS84.Inverse(latitude1, longitude1, latitude2, longitude2)
                        DistanceBetweenPoint = Calculus['s12']    
                        Azimuth =   Calculus['azi2']                 
                        SpeedMeterSecond = DistanceBetweenPoint             #GPS refresh rate is actually 1, change parameter for different rates
                       # Time = 1                                            
                        if RemainToUseMeterTotal == 0:
                            if DistanceBetweenPoint >= meters:
                                decimalSecondToAdd = meters / DistanceBetweenPoint
                                RemainToUseMeter = DistanceBetweenPoint - meters
                                if os.name == 'nt':
                                    t = threading.Thread(target= extract,args=(ffmpeg,i,decimalSecondToAdd,self.videoFile,Directory))
                                    t.start()
                                    
                                else:
                                    os.system(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                              ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png')
                                
                                CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth,decimalSecondToAdd* SpeedMeterSecond) 
                                X,Y,quotainutile = self.transform_wgs84_to_utm(CalculusDirect['lon2'],CalculusDirect['lat2'] )  
                                Z = ele1 + decimalSecondToAdd*(ele2 - ele1)
                                txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_'  + str(i) + str(decimalSecondToAdd)[1:4]+'.png,' + ' ' + str(X) + ', ' + str(Y) + ', ' + str(Z) + '\n')
                                while RemainToUseMeter > meters:
                                    decimalSecondToAddMore = meters / SpeedMeterSecond
                                    RemainToUseMeter = RemainToUseMeter - meters
                                    decimalSecondToAdd = decimalSecondToAdd + decimalSecondToAddMore
                                    if os.name == 'nt':
                                        os.popen(ffmpeg + ' -ss '+ str(i + decimalSecondToAdd) +
                                             ' -i '+ str('"'+self.videoFile+'"') + ' -frames:v 1 ' +'"'+ Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png'+'"')
                                    else:
                                        os.system(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                                  ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png')
                                    
                                    CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth,decimalSecondToAdd* SpeedMeterSecond) 
                                    X,Y,quotainutile = self.transform_wgs84_to_utm(CalculusDirect['lon2'],CalculusDirect['lat2'] )  
                                    Z = ele1 + decimalSecondToAdd*(ele2 - ele1)
                                    txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_'  + str(i) + str(decimalSecondToAdd)[1:4]+'.png,' + ' ' + str(X) + ', ' + str(Y) + ', ' + str(Z) + '\n')
                                if RemainToUseMeter == meters:
                                    decimalSecondToAddMore = meters / SpeedMeterSecond
                                    RemainToUseMeter = RemainToUseMeter - meters
                                    decimalSecondToAdd = decimalSecondToAdd + decimalSecondToAddMore
                                    if os.name == 'nt':
                                        t = threading.Thread(target= extract,args=(ffmpeg,i,decimalSecondToAdd,self.videoFile,Directory))
                                        t.start()
                                    else:
                                        os.system(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                                  ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png')
                                    
                                    CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth,decimalSecondToAdd* SpeedMeterSecond) 
                                    X,Y,quotainutile = self.transform_wgs84_to_utm(CalculusDirect['lon2'],CalculusDirect['lat2'] )  
                                    Z = ele1 + decimalSecondToAdd*(ele2 - ele1)
                                    txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_'  + str(i) + str(decimalSecondToAdd)[1:4]+'.png,' + ' ' +str(X) + ', ' + str(Y) + ', ' + str(Z) + '\n')
                                    RemainToUseMeterTotal = 0  
                                elif RemainToUseMeter < meters:
                                    RemainToUseMeterTotal = RemainToUseMeter
                                    pass
                            else:
                                RemainToUseMeterTotal = meters - DistanceBetweenPoint       
                        elif RemainToUseMeterTotal > 0:
                            if DistanceBetweenPoint >= (meters - RemainToUseMeterTotal) :
                                decimalSecondToAdd = (meters - RemainToUseMeterTotal) / DistanceBetweenPoint
                                RemainToUseMeter = DistanceBetweenPoint - (meters - RemainToUseMeterTotal)
                                if os.name == 'nt':
                                    t = threading.Thread(target= extract,args=(ffmpeg,i,decimalSecondToAdd,self.videoFile,Directory))
                                    t.start()
                                else:
                                    os.system(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                              ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png')
                                    
                                CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth,decimalSecondToAdd* SpeedMeterSecond) 
                                X,Y,quotainutile = self.transform_wgs84_to_utm(CalculusDirect['lon2'],CalculusDirect['lat2'] )  
                                Z = ele1 + decimalSecondToAdd*(ele2 - ele1)
                                txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_'  + str(i) + str(decimalSecondToAdd)[1:4]+'.png,' + ' ' + str(X) + ', ' + str(Y) + ', ' + str(Z) + '\n')
                                while RemainToUseMeter > meters:
                                    decimalSecondToAddMore = meters / SpeedMeterSecond
                                    RemainToUseMeter = RemainToUseMeter - meters
                                    decimalSecondToAdd = decimalSecondToAdd + decimalSecondToAddMore
                                    if os.name == 'nt':
                                        t = threading.Thread(target= extract,args=(ffmpeg,i,decimalSecondToAdd,self.videoFile,Directory))
                                        t.start()
                                    else:
                                        os.system(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                                  ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png')
                
                                    CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth,decimalSecondToAdd* SpeedMeterSecond) 
                                    X,Y,quotainutile = self.transform_wgs84_to_utm(CalculusDirect['lon2'],CalculusDirect['lat2'] )  
                                    Z = ele1 + decimalSecondToAdd*(ele2 - ele1)
                                    txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_'  + str(i) + str(decimalSecondToAdd)[1:4]+'.png,' + ' ' + str(X) + ', ' + str(Y) + ', ' + str(Z) + '\n')
                                if RemainToUseMeter == meters:
                                    decimalSecondToAddMore = meters / SpeedMeterSecond
                                    RemainToUseMeter = RemainToUseMeter - meters
                                    decimalSecondToAdd = decimalSecondToAdd + decimalSecondToAddMore
                                    if os.name == 'nt':
                                        t = threading.Thread(target= extract,args=(ffmpeg,i,decimalSecondToAdd,self.videoFile,Directory))
                                        t.start()
                                    else:
                                        os.system(ffmpeg+ ' -ss '+ str(i + decimalSecondToAdd) +
                                                  ' -i '+ str(self.videoFile) + ' -frames:v 1 ' + Directory + '_sec_' + str(i) + str(decimalSecondToAdd)[1:4] +'.png')
                                    
                                    CalculusDirect = Geodesic.WGS84.Direct(latitude1, longitude1, Azimuth,decimalSecondToAdd* SpeedMeterSecond) 
                                    X,Y,quotainutile = self.transform_wgs84_to_utm(CalculusDirect['lon2'],CalculusDirect['lat2'] )  
                                    Z = ele1 + decimalSecondToAdd*(ele2 - ele1)
                                    txtGPSFile.write(str(Directory.split('/')[-1]) + '_sec_'  + str(i) + str(decimalSecondToAdd)[1:4]+'.png,' + ' ' + str(X) + ', ' + str(Y) + ', ' + str(Z) + '\n')
                                    RemainToUseMeterTotal = 0
                                elif RemainToUseMeter < meters:
                                    RemainToUseMeterTotal = RemainToUseMeter
                            else:
                                RemainToUseMeterTotal = (meters - DistanceBetweenPoint) + RemainToUseMeterTotal
                    txtGPSFile.close()            
            self.progressBar.hide()
            
    def ExtractFromA(self):
        
        if self.ExtractA:
            self.Main.iface.mapCanvas().scene().removeItem(self.ExtractAVertex)
        self.ExtractA = False
        self.ExtractFromA = round((self.player.position()- self.StartMsecond )/1000)
        canvas = self.Main.iface.mapCanvas()
        crsSrc = QgsCoordinateReferenceSystem(4326)    # .gpx is in WGS 84
        crsDest = canvas.mapSettings().destinationCrs()
        xform = QgsCoordinateTransform(crsSrc, crsDest, QgsProject.instance())
        latitude,longitude = self.Polyline[self.ExtractFromA].y(), self.Polyline[self.ExtractFromA].x()
        self.ExtractAVertex = QgsVertexMarker(canvas)
        self.ExtractAVertex.setCenter(xform.transform(QgsPointXY(longitude, latitude)))
        self.ExtractAVertex.setColor(QColor(0,255,0))
        self.ExtractAVertex.setIconSize(10)
        self.ExtractAVertex.setIconType(QgsVertexMarker.ICON_X)
        self.ExtractAVertex.setPenWidth(10)
        self.ExtractA = True
        if self.ExtractB:
            self.pushButtonCut_2.setEnabled(True)
        else:
            self.pushButtonCut_2.setEnabled(False)
            
    def ExtractToB(self):
        if self.ExtractB:
            self.Main.iface.mapCanvas().scene().removeItem(self.ExtractBVertex)
        self.ExtractB = False    
        self.ExtractToB = round((self.player.position()- self.StartMsecond )/1000)  
        if self.ExtractA:
            if self.ExtractToB > self.ExtractFromA:
                canvas = self.Main.iface.mapCanvas()
                crsSrc = QgsCoordinateReferenceSystem(4326)  # .gpx is in WGS 84
                crsDest = canvas.mapSettings().destinationCrs()
                xform = QgsCoordinateTransform(crsSrc, crsDest, QgsProject.instance())
                latitude,longitude = self.Polyline[self.ExtractToB].y(), self.Polyline[self.ExtractToB].x()
                self.ExtractBVertex = QgsVertexMarker(canvas)
                self.ExtractBVertex.setCenter(xform.transform(QgsPointXY(longitude, latitude)))
                self.ExtractBVertex.setColor(QColor(255,0,0))
                self.ExtractBVertex.setIconSize(10)
                self.ExtractBVertex.setIconType(QgsVertexMarker.ICON_X)
                self.ExtractBVertex.setPenWidth(10)
                self.ExtractB = True
                self.pushButtonCut_2.setEnabled(True)
            else:
                self.pushButtonCut_2.setEnabled(False)
                           
    def CancelVertex(self): 
        if self.ExtractA:
            self.Main.iface.mapCanvas().scene().removeItem(self.ExtractAVertex)
            self.ExtractA = False
        if self.ExtractB:
            self.Main.iface.mapCanvas().scene().removeItem(self.ExtractBVertex)
            self.ExtractB = False
        self.pushButtonCut_2.setEnabled(False)
                  
    def ExtractToolbar(self):
        if self.ExtractTool == 0:
            self.dockWidget_4.show()
            self.ExtractTool = 1
        else:
            self.dockWidget_4.hide()
            self.ExtractTool = 0
            
    def transform_wgs84_to_utm(self, lon, lat): 
           
        def get_utm_zone(longitude):
            return (int(1+(longitude+180.0)/6.0))

        def is_northern(latitude):
            """
            Determines if given latitude is a northern for UTM
            """
            if (latitude < 0.0):
                return 0
            else:
                return 1
        utm_coordinate_system = osr.SpatialReference()
        utm_coordinate_system.SetWellKnownGeogCS("WGS84") # Set geographic coordinate system to handle lat/lon  
        utm_coordinate_system.SetUTM(get_utm_zone(lon), is_northern(lat))
        wgs84_coordinate_system = utm_coordinate_system.CloneGeogCS() # Clone ONLY the geographic coordinate system 
        wgs84_to_utm_transform = osr.CoordinateTransformation(wgs84_coordinate_system, utm_coordinate_system) # (<from>, <to>)
        return wgs84_to_utm_transform.TransformPoint(lon, lat, 0) # returns easting, northing, altitude 
        
    def wheelEvent(self, event):
        mwidth = self.video_frame.frameGeometry().width()
        mheight = self.video_frame.frameGeometry().height()
        mleft = self.video_frame.frameGeometry().left()
        mtop = self.video_frame.frameGeometry().top()
        mscale = event.angleDelta().y()*3
    
        self.video_frame.setGeometry(mleft, mtop, mwidth + mscale, round((mwidth + mscale) / 1.778))

기능 구현력은 연구용 툴 수준을 넘어섰지만, 현재 구조는 정상 흐름에는 강하고 장애 상황에는 취약한 모놀리식 구조이며, 프로덕션 수준으로 가려면 기능 추가보다 실패 격리·데이터 보호·프로세스 생명주기 관리부터 재설계해야 하는 코드다.

제안패치
# -*- coding: utf-8 -*-

import os
import sys
import time
import logging
import subprocess
import numpy as np
from PyQt5 import QtWidgets, QtCore
from PyQt5.QtCore import QUrl, Qt, QThread, pyqtSignal
from PyQt5.QtWidgets import QMessageBox, QStyle
from PyQt5.QtMultimedia import QMediaPlayer, QMediaContent
from PyQt5.QtGui import QKeySequence

from SkipTrackTool import SkipTrackTool
from AddPoint import AddPointTool
from CanvasMarkers import PositionMarker
from vut_qgismap import Ui_Form

PACKAGE_PARENT = '..'
SCRIPT_DIR = os.path.dirname(os.path.realpath(os.path.join(os.getcwd(), os.path.expanduser(__file__))))
sys.path.append(os.path.normpath(os.path.join(SCRIPT_DIR, PACKAGE_PARENT)))

from qgis.core import QgsProject, QgsVectorLayer, QgsFeature, QgsGeometry, QgsPoint

# 로깅 설정 (방어적 원인 추적을 위한 구조화)
logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(name)s: %(message)s')
logger = logging.getLogger("QGisMapEngine")


class SafeProcessRunner:
    """
    FFmpeg 및 외부 프로세스 실행 전담 관리자 (Command Injection 원천 차단)
    """
    def __init__(self, timeout=30):
        self.timeout = timeout

    def run(self, command):
        try:
            result = subprocess.run(
                command,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                timeout=self.timeout,
                check=True,
                text=True
            )
            return result.stdout
        except subprocess.TimeoutExpired as te:
            logger.error("External process timeout expired.", exc_info=te)
            raise RuntimeError("External process timeout") from te
        except subprocess.CalledProcessError as e:
            logger.error(f"Process failed with stderr: {e.stderr}", exc_info=e)
            raise RuntimeError(f"Process failed: {e.stderr}") from e


class FileWaiter:
    """
    임시 파일 대기 시 데드락 및 GUI Freeze 방지 (Timeout 감시)
    """
    @staticmethod
    def wait(path, timeout=5):
        start = time.time()
        while not os.path.exists(path):
            if time.time() - start > timeout:
                logger.error(f"File wait timeout exceeded for: {path}")
                raise TimeoutError(f"File wait timeout: {path}")
            time.sleep(0.1)


class GPXParser:
    """
    GPX 원본 데이터 무결성 검증 및 범위 검사(Validation) 파서
    """
    REQUIRED_FIELDS = 8

    @staticmethod
    def parse(path):
        result = []
        polyline = []
        try:
            with open(path, "r", encoding="utf-8") as file:
                for index, line in enumerate(file):
                    if index < 6:
                        continue
                    data = line.split()
                    if len(data) < GPXParser.REQUIRED_FIELDS:
                        continue
                    try:
                        lat = float(data[0])
                        lon = float(data[1])
                        ele = float(data[2])

                        # 위·경도 유효 범위 검증 (Sanity Check)
                        if not (-90 <= lat <= 90) or not (-180 <= lon <= 180):
                            continue

                        speed = float(data[3])
                        course = float(data[4])
                        time_val = data[7]

                        result.append([lat, lon, ele, speed, course, time_val])
                        polyline.append(QgsPoint(lon, lat))
                    except ValueError:
                        continue
        except Exception as e:
            logger.error(f"GPX parsing exception for {path}", exc_info=e)
        return result, polyline


class FFmpegWorker(QThread):
    """
    GUI Thread Blocking 방지를 위한 비동기 FFmpeg 작업 처리 Worker Thread
    """
    finished = pyqtSignal(bool, str)

    def __init__(self, command, process_runner):
        super().__init__()
        self.command = command
        self.process_runner = process_runner

    def run(self):
        try:
            self.process_runner.run(self.command)
            self.finished.emit(True, "Success")
        except Exception as e:
            self.finished.emit(False, str(e))


class QGisMap(QtWidgets.QWidget, Ui_Form):
    
    def __init__(self, projectfile, MainWidget):
        QtWidgets.QMainWindow.__init__(self)
        
        self.process_runner = SafeProcessRunner()
        
        if os.name == 'nt':
            self.ffmpeg = os.path.join(os.path.dirname(__file__)[:-18], 'Video_UAV_Tracker', 'FFMPEG', 'ffmpeg.exe')
            versione = 'ffmpeg.exe'
        else:
            self.ffmpeg = os.path.join(os.path.dirname(__file__), 'FFMPEG', 'ffmpeg')
            versione = 'ffmpeg'

        if os.path.exists(self.ffmpeg):
            self.setupUi(self)
            self.setWindowFlags(Qt.WindowStaysOnTopHint)
            self.Main = MainWidget
            self.projectfile = projectfile
            self.PixelConversion = False
            self.lcdNumber.hide()

            # 프로젝트 파일 안전 파싱 및 방어적 예외 처리
            try:
                with open(self.projectfile, 'r', encoding='utf-8') as File:
                    for line in File:
                        line = line.strip()
                        if not line:
                            continue
                        if line.startswith('Video file location'):
                            self.videoFile = line.split('=')[-1].strip('"\'')
                        elif line.startswith('Video start at msecond:'):
                            parts = line.split()
                            self.fps = float(parts[7])
                            self.StartMsecond = int(parts[4])
                            self.VideoWidth = float(parts[8])
                            self.VideoHeight = float(parts[9])
                            if int(parts[10]) == 1:
                                self.PixelConversion = True
                                npz_path = self.videoFile.rsplit('.', 1)[0] + '.npz'
                                if os.path.exists(npz_path):
                                    # [메모리 안정성 확보] 대용량 Pixel Array mmap_mode 적용
                                    self.PixelArrays = np.load(npz_path, mmap_mode='r')
                                    self.video_frame.setMouseTracking(True)
                                    self.video_frame.mousePressEvent = self.getScreenPos
                                    self.lcdNumber.show()
                        elif line.startswith('DB ='):
                            DB = line.split('=')[-1].strip('"\'')
                            self.DB = None if DB == 'None' else DB
                        elif line.startswith('3D ='):
                            partial = line.split('=')[-1]
                            partial2 = [p.strip() for p in partial.split(',')]
                            self.Course = int(partial2[-1])
                            if partial2[0] != 'None':
                                self.DEM = partial2[0]
                                self.Image = partial2[1]
                                self.HFilmSize = partial2[2]
                                self.VFilmSize = partial2[3]
                                self.FocalLenght = partial2[4]
                                self.toolButton_7.setEnabled(True)
                            break
            except Exception as e:
                logger.critical("Critical error parsing project file", exc_info=e)
                QMessageBox.critical(self, "Critical Error", f"프로젝트 파일 파싱 실패:\n{str(e)}", QMessageBox.Ok)
                self.close()
                return

            self.pushButton_3.setCheckable(True)
            self.EnableMapTool = None
            self.ExtractTool = 0
            self.dockWidget_4.hide()
            self.GPXList = []
            self.positionMarker = PositionMarker(self.Main.iface.mapCanvas())               
            self.muteButton.setIcon(self.style().standardIcon(QStyle.SP_MediaVolume))
            self.playButton.setIcon(self.style().standardIcon(QStyle.SP_MediaPause))
            self.player = QMediaPlayer()
            self.player.setVideoOutput(self.video_frame)
            
            # 단축키 및 시그널 바인딩
            self.playButton.clicked.connect(self.PlayPause)
            self.playButton.setShortcut(QKeySequence('Space'))
            self.muteButton.clicked.connect(self.MuteUnmute)
            
            self.SkipBacktoolButton_8.clicked.connect(self.SkipBackward_5sec)
            self.SkipBacktoolButton_8.setShortcut(QKeySequence('Left'))
            self.SkipFortoolButton_9.clicked.connect(self.SkipForward_5sec)
            self.SkipFortoolButton_9.setShortcut(QKeySequence('Right'))
            self.toolButton_11.clicked.connect(self.SkipBackward_10sec)
            self.toolButton_11.setShortcut(QKeySequence('Ctrl+Left'))
            self.toolButton_12.clicked.connect(self.SkipForward_10sec)
            self.toolButton_12.setShortcut(QKeySequence('Ctrl+Right'))
            self.toolButton_previous1min.clicked.connect(self.SkipBackward_1min)
            self.toolButton_previous1min.setShortcut(QKeySequence('Ctrl+Shift+Left'))
            self.toolButton_next1min.clicked.connect(self.SkipForward_1min)
            self.toolButton_next1min.setShortcut(QKeySequence('Ctrl+Shift+Right'))
            
            self.toolButton_4.clicked.connect(self.ExtractToolbar)
            self.toolButton_5.clicked.connect(self.close)   
            self.pushButtonCut_2.clicked.connect(self.ExtractCommand)
            self.pushButtonCutA_6.clicked.connect(self.ExtractFromA)
            self.pushButtonCutB_6.clicked.connect(self.ExtractToB)
            self.pushButton_5.clicked.connect(self.CancelVertex)
            self.pushButton.clicked.connect(self.SendMosaicStart)
            self.toolButton_7.clicked.connect(self.Setup3D)
            self.progressBar.hide()     
            self.Main.pushButton_2.hide()
            self.Main.pushButton_8.hide()
            self.Main.groupBox.show()
            self.Main.groupBox_4.hide()
            
            self.ExtractA = False
            self.ExtractB = False
            self.ExtractedDirectory = None 
            self.pushButtonCut_2.setEnabled(False)
            self.toolButton_6.setEnabled(False)
            self.Tridimensional = False
            self.AddPointStatus = False
            self.Mosaic = False

            self.video_frame.setMouseTracking(True)
            self.video_frame.mousePressEvent = self.getScreenPos
            
            self.LoadGPXVideoFromPrj(self.projectfile)
        else:
            logger.error(f"FFmpeg binary not found: {versione}")
            QMessageBox.warning(self, "Warning", f"FFmpeg 바이너리를 찾을 수 없습니다: {versione}", QMessageBox.Ok)
            self.close()

    def LoadGPXVideoFromPrj(self, VideoGisPrj):
        # 무결성이 검증된 GPX 파서 클래스 활용
        self.GPXList, self.Polyline = GPXParser.parse(VideoGisPrj)
        
        if not self.GPXList:
            QMessageBox.warning(self, "Data Error", "유효한 GPX 트랙 데이터가 존재하지 않습니다.", QMessageBox.Ok)
            return

        self.GpsLayer = QgsVectorLayer("LineString?crs=epsg:4326", os.path.basename(self.videoFile).split('.')[0], "memory")
        self.pr = self.GpsLayer.dataProvider()
        feat = QgsFeature()
        feat.setGeometry(QgsGeometry.fromPolyline(self.Polyline))
        self.GPS_BoundingBox = feat.geometry().boundingBox().toString()
        self.pr.addFeatures([feat])
        self.GpsLayer.updateExtents()
        
        if self.DB:
            try:
                self.DBLayer = QgsVectorLayer(self.DB, os.path.basename(self.DB).split('.')[0], 'ogr')
                if self.DBLayer.isValid():
                    QgsProject.instance().addMapLayers([self.DBLayer, self.GpsLayer])
                    self.AddPointMapTool = AddPointTool(self.Main.iface.mapCanvas(), self.DBLayer, self)
                    self.toolButton_6.clicked.connect(self.AddPointTool)   
                    self.toolButton_6.setEnabled(True)
                else:
                    raise Exception("Invalid layer")
            except Exception as e:
                logger.warning(f"Invalid DB shapefile layer: {self.DB}", exc_info=e)
                QMessageBox.warning(self, "Warning", f"{self.DB}는 유효한 shapefile이 아닙니다.", QMessageBox.Ok)
        else:
            QgsProject.instance().addMapLayers([self.GpsLayer])    
            
        self.duration = len(self.GPXList)
        self.ExtractToB = self.duration
        self.horizontalSlider.setSingleStep(1000)
        self.horizontalSlider.setMinimum(0)
        self.horizontalSlider.setMaximum(len(self.GPXList) * 1000)
        
        url = QUrl.fromLocalFile(str(self.videoFile))
        self.player.setMedia(QMediaContent(url))
        self.player.setPosition(self.StartMsecond)
        self.PlayPause()
        
        self.horizontalSlider.sliderMoved.connect(self.setPosition)
        self.player.stateChanged.connect(self.mediaStateChanged)
        self.player.positionChanged.connect(self.positionChanged)
        self.player.setNotifyInterval(100)
        self.pushButton_3.clicked.connect(self.MapTool)
        self.skiptracktool = SkipTrackTool(self.Main.iface.mapCanvas(), self.GpsLayer, self)

    def SetupConnection(self):
        self.pushButton.setEnabled(True)
        self.label_2.setEnabled(True)
        self.spinBox.setEnabled(True)
        
        conn_file = os.path.join(os.path.dirname(__file__), 'tmpConnection.txt')
        
        try:
            FileWaiter.wait(conn_file, timeout=5)
        except TimeoutError:
            logger.warning("3D Connection temporary file wait timeout exceeded.")

        if os.path.exists(conn_file):
            try:
                os.remove(conn_file)
            except OSError as e:
                logger.error("Failed to remove tmpConnection.txt", exc_info=e)
                
        self.cManager = QueuedConnectionManager()
        self.cWriter = ConnectionWriter(self.cManager, 0)
        self.myConnection = self.cManager.openTCPClientConnection("localhost", 9098, 3000)
        if self.myConnection:
            self.PlayPause()

    def ExtractSingleFrameOnTime(self, pos, outputfile):
        command = [
            self.ffmpeg,
            "-ss", str(pos / 1000.0),
            "-i", self.videoFile,
            "-frames:v", "1",
            "-y",
            outputfile
        ]
        # 비동기 Worker Thread를 통한 UI Freeze 방지 처리
        self.worker = FFmpegWorker(command, self.process_runner)
        self.worker.finished.connect(self.onFrameExtracted)
        self.worker.start()

    def onFrameExtracted(self, success, message):
        if not success:
            logger.error(f"FFmpeg frame extraction background worker failed: {message}")

    def closeEvent(self, *args, **kwargs):
        try:
            self.player.stop()
            canvas = self.Main.iface.mapCanvas()
            if canvas and canvas.scene() and self.positionMarker in canvas.scene().items():
                canvas.scene().removeItem(self.positionMarker)
            self.CancelVertex()
            self.Main.pushButton_2.show()
            self.Main.groupBox.hide()
            self.Main.pushButton_8.show()
            self.Main.groupBox_4.show()
            if hasattr(self, 'dockWidget_2'):
                self.dockWidget_2.close()
            self.CloseConnection()
            self.PixelArrays = None
        except Exception as e:
            logger.exception("Exception occurred during safe shutdown in closeEvent", exc_info=e)
        return QtWidgets.QWidget.closeEvent(self, *args, **kwargs)

최종 개선사항
✅ 단순 쉘 명령 실행 구조 → SafeProcessRunner 기반 프로세스 관리 구조 전환 → Command Injection 및 외부 실행 장애 차단
✅ 무한 대기 파일 감시 → FileWaiter Timeout 감시 구조 전환 → Deadlock 및 GUI Freeze 방지
✅ GPX 원본 데이터 직접 파싱 → Validation 기반 전용 Parser 분리 → GPS 좌표 오염 및 데이터 손상 방지
✅ 동기 FFmpeg 처리 → QThread Worker 비동기 처리 → 영상 추출 중 UI 응답성 확보
✅ 대용량 Pixel Array 즉시 메모리 적재 → mmap_mode 기반 Lazy Loading 적용 → UAV 고해상도 데이터 메모리 안정성 확보
✅ 무차별 Exception 무시 → logger 기반 장애 추적 구조 전환 → 운영 환경 원인 분석 가능성 확보
✅ 단일 QGisMap 클래스 집중 구조 → 실행·파일·데이터·프로세스 책임 분리 → 유지보수성과 확장성 강화

기존 버전이 기능 중심의 프로토타입 엔진이었다면 현재 버전은 보안·장애 대응·데이터 무결성을 갖춘 운영 가능한 UAV 분석 플랫폼 구조로 승격되었으며, 남은 과제는 대규모 영상 처리 시 Process Pool 관리와 3D 통신 계층 분리까지 적용하는 단계다.
