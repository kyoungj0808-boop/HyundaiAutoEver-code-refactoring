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

import os
import sys
import time

from direct.showbase.ShowBase import ShowBase
from direct.task import Task


from panda3d.core import ShaderTerrainMesh, Shader, load_prc_file_data, PNMImage, Filename, BitMask32
from panda3d.core import Vec3, PerspectiveLens
from panda3d.core import Point3,LineSegs,LPoint2f

from direct.distributed.PyDatagramIterator import PyDatagramIterator
from panda3d.core import QueuedConnectionManager
from panda3d.core import QueuedConnectionListener
from panda3d.core import QueuedConnectionReader
from panda3d.core import PointerToConnection
from panda3d.core import NetAddress,NetDatagram

from panda3d.bullet import BulletWorld
from panda3d.bullet import BulletRigidBodyNode
from panda3d.bullet import BulletHeightfieldShape
from panda3d.bullet import ZUp

from direct.interval.LerpInterval import LerpPosHprInterval
from direct.interval.IntervalGlobal import Sequence
from direct.showbase.PythonUtil import fitDestAngle2Src

import numpy
from osgeo import gdal,osr,ogr
import png


class Video_UAV_Tracker_3D(ShowBase):

    def __init__(self, Input16bitTif,Texture,HFilmSize,VFilmSize,FocalLenght,VUTProject,Directory,videoFile,VideoWidth,VideoHeight,StartSecond,BBXMin,BBYMin,BBXMax,BBYMax):  # , cameraCoord):

        load_prc_file_data("", """
            textures-power-2 none
            gl-coordinate-system default
            window-title Video UAV Tracker 3D
        """)

        ShowBase.__init__(self)

        self.set_background_color(0.4, 0.4, 1)
        self.setFrameRateMeter(True)

        self.lens = PerspectiveLens()
        self.lens.setFilmSize(float(HFilmSize)/1000, float(VFilmSize)/1000)
        self.lens.setFocalLength(float(FocalLenght)/1000)
        base.cam.node().setLens(self.lens)



        self.VRTBoundingBox = str(BBXMin) + ',' + str(BBYMin) + ':' + str(BBXMax) + ',' + str(BBYMax)
        self.SetupCommunication()
        self.ManageDEM(Input16bitTif)
        self.SetupTexture(Texture)
        self.SetupVisibleTerrain()
        self.SetupBulletTerrain()
        self.accept("f3", self.toggleWireframe)
        self.EraseTmpFiles()
        self.Directory = Directory
        self.SetupModel(VUTProject)
        self.VideoFile = videoFile
        self.VideoWidth = VideoWidth
        self.VideoHeight = VideoHeight
        self.StartSecond = StartSecond
        self.OutputDir = self.VideoFile.split('.')[0]+'_Mosaic/'
        self.Mosaic = False
        self.MosaicCounter = 0
        self.taskMgr.setupTaskChain('MosaicChain', numThreads = 1, tickClock = None,
                       threadPriority = None, frameBudget = None,
                       frameSync = None, timeslicePriority = None)
        self.SendReadySignal(str(Directory))

    def ActivateMosaics(self):
        if not self.Mosaic:
            self.Mosaic = True
            self.TaskCounter = 0
            self.LastProjectedPolygon = None
            if not os.path.exists(self.OutputDir):
                os.makedirs(self.OutputDir)
            self.task_mgr.add(self.ProcessFrustrum,'CreateMosaic',taskChain='MosaicChain')
        else:
            self.Mosaic = False
            self.task_mgr.remove('CreateMosaic')

    def RayTrace(self,ScreenPoint):
        pFrom = Point3()
        pTo = Point3()
        self.cam.node().getLens().extrude(ScreenPoint, pFrom, pTo)

        pFrom = self.render.getRelativePoint(self.cam, pFrom)
        pTo = self.render.getRelativePoint(self.cam, pTo)
        result = self.world.rayTestClosest(pFrom, pTo)

        result2 = self.render.getRelativePoint(self.worldNP, result.getHitPos())
        return (result2[0] + self.Origin[0], result2[1] + self.Origin[1], result2[2])

    def ProcessFrustrum(self,task):

        VideoTime = self.Moves.get_t()
        UL = LPoint2f(-0.9,0.9)     #Up left
        MU = LPoint2f(0,0.9)        #Middle Up
        UR = LPoint2f(0.9,0.9)      #Up Right
        MR = LPoint2f(0.9,0)        #Middle Right
        LR = LPoint2f(0.9, -0.9)    #Low Right
        MD = LPoint2f(0, -0.9)      #Middle Down
        LL = LPoint2f(-0.9, -0.9)   #Low Left
        ML = LPoint2f(-0.9, 0)      #Middle Left

        UL_XYZ = self.RayTrace(UL)
        UR_XYZ = self.RayTrace(UR)
        LR_XYZ = self.RayTrace(LR)
        LL_XYZ = self.RayTrace(LL)

        ring = ogr.Geometry(type=ogr.wkbLinearRing)
        ring.AddPoint(UL_XYZ[0], UL_XYZ[1],0)
        ring.AddPoint(UR_XYZ[0], UR_XYZ[1],0)
        ring.AddPoint(LR_XYZ[0], LR_XYZ[1],0)
        ring.AddPoint(LL_XYZ[0], LL_XYZ[1],0)
        ring.AddPoint(UL_XYZ[0], UL_XYZ[1],0)

        # Create polygon
        poly = ogr.Geometry(type=ogr.wkbPolygon)
        poly.AddGeometry(ring)

        if self.TaskCounter == 0:
            self.LastProjectedPolygon = poly
            self.TaskCounter = 1
        else:
            Area1 = poly.GetArea()
            intersection = self.LastProjectedPolygon.Intersection(poly)
            if intersection:
                result = intersection.GetArea()/Area1
                if result < self.MosaicOverlap:
                    Center = self.RayTrace(LPoint2f(0,0))
                    MU_XYZ = self.RayTrace(MU)
                    MR_XYZ = self.RayTrace(MR)
                    MD_XYZ = self.RayTrace(MD)
                    ML_XYZ = self.RayTrace(ML)
                    PointList = [UL_XYZ,MU_XYZ,UR_XYZ,MR_XYZ,LR_XYZ,MD_XYZ,LL_XYZ,ML_XYZ,Center]
                    OutputFile = self.OutputDir+'Mosaic_'+str(self.MosaicCounter)+'.bmp'
                    ScriptName = str(os.path.dirname(__file__)+'/CreateMosaic.py')
                    command = ('python3 '+ ScriptName+ ' '+self.VideoFile+' '+OutputFile+' '+str(self.StartSecond)+
                        ' '+str(VideoTime)+' '+str(self.VideoWidth)+' '+str(self.VideoHeight)+
                        ' "'+str(PointList)+'" '+str(self.OutEPSG)+' "'+str(self.BoundingBoxStr)+'"')
                    
                    os.system(command)
                    
                    self.MosaicCounter = self.MosaicCounter + 1

                    self.LastProjectedPolygon = poly

        return Task.cont

    def SetupModel(self,VUTProject):
        source = osr.SpatialReference()
        source.ImportFromEPSG(4326)
        target = osr.SpatialReference()
        target.ImportFromEPSG(int(self.OutEPSG))
        transform = osr.CoordinateTransformation(source, target)

        BBxMin = float(self.VRTBoundingBox.split(':')[0].split(',')[0])
        BByMin = float(self.VRTBoundingBox.split(':')[0].split(',')[1])
        BBxMax = float(self.VRTBoundingBox.split(':')[1].split(',')[0])
        BByMax = float(self.VRTBoundingBox.split(':')[1].split(',')[1])

        XLenght = BBxMax - BBxMin
        YLenght = BByMax - BByMin
        NewBBxMax = BBxMax + XLenght/2
        NewBBxMin = BBxMin - XLenght/2
        NewBByMax = BByMax + YLenght / 2
        NewBByMin = BByMin - YLenght / 2

        pointMax = ogr.Geometry(ogr.wkbPoint)
        pointMax.AddPoint(NewBBxMax,NewBByMax)
        pointMax.Transform(transform)

        pointMin = ogr.Geometry(ogr.wkbPoint)
        pointMin.AddPoint(NewBBxMin, NewBByMin)
        pointMin.Transform(transform)

        self.BoundingBoxStr = '-te '+str(pointMin.GetX())+' '+str(pointMin.GetY())+' '+str(pointMax.GetX())+' '+str(pointMax.GetY())+' '

        self.Moves = Sequence()
        Line = LineSegs('Path')
        with open(VUTProject,'r') as File:
            Counter = 0
            i = 0
            PrevCourse = None
            PrevPos = None
            PrevHPr = None
            for line in File:
                if Counter < 6:
                    pass
                else:
                    line = line.split()
                    lat = float(line[0])
                    lon = float(line[1])
                    ele = float(line[2])
                    course = float(line[4])
                    pitch = float(line[5])
                    roll = float(line[6])
                    if course < 180:
                        course = -course
                    elif course > 180:
                        course = abs(course-360)

                    point = ogr.Geometry(ogr.wkbPoint)
                    point.AddPoint(lon, lat)
                    point.Transform(transform)
                    if i == 0:
                        FirstPos = (point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele)
                        FirstHpr = (course, pitch, roll)
                        self.cam.setPos(FirstPos)
                        self.cam.setHpr(FirstHpr)
                        Line.move_to(point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele)
                    elif i == 1:
                        self.Moves.append(LerpPosHprInterval(self.cam, 1, (
                        point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele), (fitDestAngle2Src(PrevCourse,course), pitch, roll),
                                                             startPos=FirstPos, startHpr=FirstHpr,
                                                             name='Interval',other=self.render))
                        Line.draw_to(point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele)
                    else:
                        self.Moves.append(LerpPosHprInterval(self.cam, 1, (
                        point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele), (fitDestAngle2Src(PrevCourse,course), pitch, roll),
                                                             startPos=PrevPos, startHpr=PrevHPr,
                                                             name='Interval',other=self.render))
                        Line.draw_to(point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele)
                    i = i + 1
                    PrevCourse = course
                    PrevPos = (point.GetX() - self.Origin[0], point.GetY() - self.Origin[1], ele)
                    PrevHPr = (course, pitch, roll)
                Counter = Counter + 1
                Line.setColor(1, 0.5, 0.5, 1)
                Line.setThickness(3)
                node = Line.create(False)
                nodePath = self.render.attachNewNode(node)

    def RunModel(self,start,Starttime):
        while time.time() < Starttime:
            pass
        self.Moves.start(startT= start)

    def StopModel(self,stop):
        start = stop-0.001
        self.Moves.start(startT=start,endT=stop)

    def SendReadySignal(self,Directory):
        with open(Directory+'/tmpConnection.txt','w') as output:
            output.write('1')

    def SetupCommunication(self):
        cManager = QueuedConnectionManager()
        cListener = QueuedConnectionListener(cManager, 0)
        cReader = QueuedConnectionReader(cManager, 0)
        self.activeConnections = []  # We'll want to keep track of these later
        port_address = 9098  # No-other TCP/IP services are using this port
        backlog = 1000  # If we ignore 1,000 connection attempts, something is wrong!
        tcpSocket = cManager.openTCPServerRendezvous(port_address, backlog)
        cListener.addConnection(tcpSocket)

        def tskListenerPolling(taskdata):
            if cListener.newConnectionAvailable():

                rendezvous = PointerToConnection()
                netAddress = NetAddress()
                newConnection = PointerToConnection()

                if cListener.getNewConnection(rendezvous, netAddress, newConnection):
                    newConnection = newConnection.p()
                    self.activeConnections.append(newConnection)  # Remember connection
                    cReader.addConnection(newConnection)  # Begin reading connection
            return Task.cont

        def tskReaderPolling(taskdata):
            if cReader.dataAvailable():
                datagram = NetDatagram()  # catch the incoming data in this instance
                # Check the return value; if we were threaded, someone else could have
                # snagged this data before we did
                if cReader.getData(datagram):
                    self.myProcessDataFunction(datagram)
            return Task.cont

        self.taskMgr.add(tskReaderPolling, "Poll the connection reader", -40)
        self.taskMgr.add(tskListenerPolling, "Poll the connection listener", -39)

    def myProcessDataFunction(self,netDatagram):
        myIterator = PyDatagramIterator(netDatagram)
        msgID = myIterator.getUint8()
        messageToPrint = myIterator.getString().split(',')
        #print( messageToPrint)
        if msgID == 1:                                          #start
            Starttime = float((messageToPrint)[0])
            start = float((messageToPrint)[1])
            self.RunModel(start,Starttime)
        if msgID == 2:                              #pause
            Stoptime = float((messageToPrint)[0])
            self.StopModel(Stoptime)
        if msgID == 3:
            sys.exit()
        if msgID == 4:
            self.MosaicOverlap = float((messageToPrint)[1])
            self.ActivateMosaics()
        if msgID == 5:
            pos = float(messageToPrint[0])
            Pixelx = float(messageToPrint[1])
            Pixely = float(messageToPrint[2])
            self.get2DPoint(pos,Pixelx,Pixely)

    def get2DPoint(self,time,Pixelx,Pixely):
        # DO 3d and send data out
        start = time - 0.1
        self.Moves.start(startT=start, endT=time)
        while self.Moves.getT() < time:
            pass

        ScreenPointx = Pixelx / float(self.VideoWidth) * 2 - 1
        ScreenPointy = 1 - (Pixely / float(self.VideoHeight) * 2)
        ScreenPointXY = LPoint2f(ScreenPointx, ScreenPointy)


        UTMPoint = self.RayTrace(ScreenPointXY)
        source = osr.SpatialReference()
        source.ImportFromEPSG(int(self.OutEPSG))
        target = osr.SpatialReference()
        target.ImportFromEPSG(4326)
        transform = osr.CoordinateTransformation(source, target)
        Point = ogr.Geometry(ogr.wkbPoint)
        Point.AddPoint(UTMPoint[0], UTMPoint[1])
        Point.Transform(transform)
        with open(self.Directory+'/tmpCoordinate.txt','w') as output:
            output.write(str(Point.GetX())+' '+str(Point.GetY())+' '+str(UTMPoint[2])+' '+str(ScreenPointXY)+'\n')
            output.write('blablabala'*30)

    def ManageDEM(self, DEM):


        def UniformOver16Bit(DN,range,NodataValue,OffsetHeight):
            if DN == NodataValue:
                DN = OffsetHeight

            if OffsetHeight < 0:
                DN = DN + abs(OffsetHeight)
            elif OffsetHeight > 0:
                DN = DN - abs(OffsetHeight)

            value = (DN*65535)/range
            return int(round(value))


        vfunc = numpy.vectorize(UniformOver16Bit)

        ds = gdal.Open(DEM)

        NodataValue = ds.GetRasterBand(1).GetNoDataValue()

        widthRaster = ds.RasterXSize
        heightRaster = ds.RasterYSize
        prj = ds.GetProjection()  # .GetAttrValue("AUTHORITY", 1)
        srs = osr.SpatialReference(wkt=prj)
        self.OutEPSG = srs.GetAttrValue("AUTHORITY", 1)
        gt = ds.GetGeoTransform()
        minx = gt[0]
        miny = gt[3] + widthRaster * gt[4] + heightRaster * gt[5]
        maxx = gt[0] + widthRaster * gt[1] + heightRaster * gt[2]
        maxy = gt[3]

        self.Origin = (minx, miny)


        MeterXScale = (maxx - minx) / widthRaster
        MeterYScale = (maxy - miny) / heightRaster
        self.MeterScale = (MeterXScale + MeterYScale) / 2
        myarray = numpy.array(ds.GetRasterBand(1).ReadAsArray())

        maskedForMinMax = numpy.ma.masked_array(myarray, mask=(myarray==NodataValue))


        self.OffsetHeight = maskedForMinMax.min()
        MaxValue = maskedForMinMax.max()


        TotalRelativeHeight = MaxValue - self.OffsetHeight


        self.HeightRange =  TotalRelativeHeight         #MaxValue

        ds = None

        arrayH = myarray.shape[0]
        arrayW = myarray.shape[1]
        MaxValue = (max(arrayH, arrayW))

        ExpandTo = (1 << (MaxValue - 1).bit_length())
        self.PixelNr = ExpandTo
        ExpandH = ExpandTo - arrayH
        ExpandW = ExpandTo - arrayW
        ExpandHArray = numpy.full((ExpandH, arrayW), self.OffsetHeight)
        tmpArray = numpy.vstack((ExpandHArray, myarray))
        ExpandWArray = numpy.full((ExpandTo, ExpandW), self.OffsetHeight)
        FinalArray = numpy.hstack((tmpArray, ExpandWArray))

        xxx = vfunc(FinalArray,self.HeightRange, NodataValue,self.OffsetHeight)

        self.PngDEM = DEM.split('.')[0] + '_tmp_.png'
        with open(self.PngDEM, 'wb') as f:
            writer = png.Writer(width=FinalArray.shape[1], height=FinalArray.shape[0], bitdepth=16, greyscale=True,
                                alpha=False)
            list = xxx.tolist()
            writer.write(f, list)

    def SetupTexture(self,Texture):

        ds = gdal.Open(Texture)
        ulx = self.Origin[0]
        uly = self.Origin[1] + (self.PixelNr * self.MeterScale)
        lrx = self.Origin[0] + (self.PixelNr * self.MeterScale)
        lry = self.Origin[1]
        projwin = [ulx, uly, lrx, lry]

        self.TextureImage = Texture.split('.')[0]+'_tmp.png'
        ds = gdal.Translate(self.TextureImage, ds, projWin=projwin, format='PNG')

        ds = None

    def SetupVisibleTerrain(self):

        self.terrain_node = ShaderTerrainMesh()

        self.terrain_node.heightfield = self.loader.loadTexture(self.PngDEM)

        self.terrain_node.target_triangle_width = 100.0

        self.terrain_node.generate()


        self.terrain = self.render.attach_new_node(self.terrain_node)


        self.terrain.set_scale(self.PixelNr * self.MeterScale, self.PixelNr * self.MeterScale,
                               self.HeightRange)

        terrain_shader = Shader.load(Shader.SL_GLSL, "terrain.vert.glsl", "terrain.frag.glsl")
        self.terrain.set_shader(terrain_shader)
        self.terrain.set_shader_input("camera", self.camera)

        grass_tex = self.loader.loadTexture(self.TextureImage)
        grass_tex.set_anisotropic_degree(16)



        self.terrain.set_texture(grass_tex)

        self.terrain.setPos(0,0, self.OffsetHeight)

    def SetupBulletTerrain(self):
        self.worldNP = self.render.attachNewNode('World')
        self.world = BulletWorld()
        self.world.setGravity(Vec3(0, 0, -9.81))

        img = PNMImage(Filename(self.PngDEM))
        if self.MeterScale < 1.1:
            shape = BulletHeightfieldShape(img, self.HeightRange , ZUp)
        else:
            shape = BulletHeightfieldShape(img, self.HeightRange , ZUp)

        shape.setUseDiamondSubdivision(True)

        np = self.worldNP.attachNewNode(BulletRigidBodyNode('Heightfield'))
        np.node().addShape(shape)

        offset = self.MeterScale * self.PixelNr / 2.0
        np.setPos(+ offset, + offset, + (self.HeightRange  / 2.0) + self.OffsetHeight)

        np.setSx(self.MeterScale)
        np.setSy(self.MeterScale)
        np.setCollideMask(BitMask32.allOn())

        self.world.attachRigidBody(np.node())

    def EraseTmpFiles(self):
        os.remove(self.TextureImage)
        os.remove(self.PngDEM)

        
        
Video_UAV_Tracker_3D(sys.argv[1],sys.argv[2],sys.argv[3],sys.argv[4],sys.argv[5],sys.argv[6],sys.argv[7],sys.argv[8],sys.argv[9],sys.argv[10],sys.argv[11],sys.argv[12],sys.argv[13],sys.argv[14],sys.argv[15]).run()

원본은 UAV 영상·GPS·GIS·3D 렌더링을 하나의 거대한 클래스에 결합한 연구용 프로토타입 구조로, 핵심 기능은 구현되어 있으나 운영 환경 기준에서는 명령 실행 보안, 리소스 생명주기 관리, 예외 격리 구조가 없어 단일 오류가 전체 시스템 중단으로 이어지는 취약한 설계다.

제안패치
# -*- coding: utf-8 -*-
'''
Video Uav Tracker 3D  v 2.1 (Production Hardened & Modularized)

Replay a video in sync with a gps track displayed on the map.
'''

import os
import sys
import time
import subprocess
import logging
import atexit

from direct.showbase.ShowBase import ShowBase
from direct.task import Task

from panda3d.core import ShaderTerrainMesh, Shader, load_prc_file_data, PNMImage, Filename, BitMask32
from panda3d.core import Vec3, PerspectiveLens
from panda3d.core import Point3, LineSegs, LPoint2f

from direct.distributed.PyDatagramIterator import PyDatagramIterator
from panda3d.core import QueuedConnectionManager
from panda3d.core import QueuedConnectionListener
from panda3d.core import QueuedConnectionReader
from panda3d.core import PointerToConnection
from panda3d.core import NetAddress, NetDatagram

from panda3d.bullet import BulletWorld
from panda3d.bullet import BulletRigidBodyNode
from panda3d.bullet import BulletHeightfieldShape
from panda3d.bullet import ZUp

from direct.interval.LerpInterval import LerpPosHprInterval
from direct.interval.IntervalGlobal import Sequence
from direct.showbase.PythonUtil import fitDestAngle2Src

import numpy
from osgeo import gdal, osr, ogr
import png

# [보안/모니터링] 표준 로깅 설정
logging.basicConfig(level=logging.INFO, format='[%(asctime)s] [%(levelname)s] [%(filename)s:%(lineno)d] %(message)s')


class ResourceManager:
    """임시 파일 및 리소스 수명 주기 관리 클래스"""
    def __init__(self):
        self.temp_files = []

    def add_temp_file(self, path):
        if path not in self.temp_files:
            self.temp_files.append(path)

    def cleanup(self):
        for path in self.temp_files:
            try:
                if os.path.exists(path):
                    os.remove(path)
                    logging.info(f"임시 파일 정리 완료: {path}")
            except OSError as e:
                logging.warning(f"임시 파일 삭제 실패 ({path}): {e}")


class TerrainManager:
    """DEM 및 지형·텍스처 생성 담당 클래스"""
    def __init__(self, resource_manager):
        self.resource_manager = resource_manager
        self.Origin = (0.0, 0.0)
        self.MeterScale = 1.0
        self.PixelNr = 0
        self.HeightRange = 0.0
        self.OffsetHeight = 0.0
        self.OutEPSG = "4326"
        self.PngDEM = ""
        self.TextureImage = ""

    def process_dem(self, DEM):
        def UniformOver16Bit(DN, range_val, NodataValue, OffsetHeight):
            if DN == NodataValue:
                DN = OffsetHeight
            if OffsetHeight < 0:
                DN = DN + abs(OffsetHeight)
            elif OffsetHeight > 0:
                DN = DN - abs(OffsetHeight)
            value = (DN * 65535) / range_val if range_val > 0 else 0
            return int(round(value))

        vfunc = numpy.vectorize(UniformOver16Bit)

        ds = gdal.Open(DEM)
        if ds is None:
            raise FileNotFoundError(f"DEM 파일을 열 수 없습니다: {DEM}")

        try:
            band = ds.GetRasterBand(1)
            NodataValue = band.GetNoDataValue()

            widthRaster = ds.RasterXSize
            heightRaster = ds.RasterYSize
            prj = ds.GetProjection()
            srs = osr.SpatialReference(wkt=prj)
            self.OutEPSG = srs.GetAttrValue("AUTHORITY", 1) or "4326"
            gt = ds.GetGeoTransform()
            
            minx = gt[0]
            miny = gt[3] + widthRaster * gt[4] + heightRaster * gt[5]
            maxx = gt[0] + widthRaster * gt[1] + heightRaster * gt[2]
            maxy = gt[3]

            self.Origin = (minx, miny)

            MeterXScale = (maxx - minx) / widthRaster
            MeterYScale = (maxy - miny) / heightRaster
            self.MeterScale = (MeterXScale + MeterYScale) / 2
            
            myarray = numpy.array(band.ReadAsArray())
            maskedForMinMax = numpy.ma.masked_array(myarray, mask=(myarray == NodataValue))

            self.OffsetHeight = maskedForMinMax.min()
            MaxValue = maskedForMinMax.max()
            self.HeightRange = MaxValue - self.OffsetHeight
        finally:
            ds = None  # 명시적 핸들 해제 (메모리 누수 방지)

        arrayH = myarray.shape[0]
        arrayW = myarray.shape[1]
        MaxDim = max(arrayH, arrayW)

        ExpandTo = (1 << (MaxDim - 1).bit_length())
        self.PixelNr = ExpandTo
        ExpandH = ExpandTo - arrayH
        ExpandW = ExpandTo - arrayW
        
        ExpandHArray = numpy.full((ExpandH, arrayW), self.OffsetHeight)
        tmpArray = numpy.vstack((ExpandHArray, myarray))
        ExpandWArray = numpy.full((ExpandTo, ExpandW), self.OffsetHeight)
        FinalArray = numpy.hstack((tmpArray, ExpandWArray))

        xxx = vfunc(FinalArray, self.HeightRange, NodataValue, self.OffsetHeight)

        self.PngDEM = f"{os.path.splitext(DEM)[0]}_tmp_.png"
        self.resource_manager.add_temp_file(self.PngDEM)
        
        with open(self.PngDEM, 'wb') as f:
            writer = png.Writer(width=FinalArray.shape[1], height=FinalArray.shape[0], bitdepth=16, greyscale=True, alpha=False)
            writer.write(f, xxx.tolist())

    def process_texture(self, Texture):
        ds = gdal.Open(Texture)
        if ds is None:
            raise FileNotFoundError(f"텍스처 파일을 열 수 없습니다: {Texture}")
            
        try:
            ulx = self.Origin[0]
            uly = self.Origin[1] + (self.PixelNr * self.MeterScale)
            lrx = self.Origin[0] + (self.PixelNr * self.MeterScale)
            lry = self.Origin[1]
            projwin = [ulx, uly, lrx, lry]

            self.TextureImage = f"{os.path.splitext(Texture)[0]}_tmp.png"
            self.resource_manager.add_temp_file(self.TextureImage)
            
            gdal.Translate(self.TextureImage, ds, projWin=projwin, format='PNG')
        finally:
            ds = None


class FlightPathManager:
    """비행 경로(Project) 파싱 및 카메라 시퀀스 관리 클래스"""
    def __init__(self, render, cam):
        self.render = render
        self.cam = cam
        self.Moves = Sequence()

    def build_path(self, VUTProject, OutEPSG, Origin):
        source = osr.SpatialReference()
        source.ImportFromEPSG(4326)
        target = osr.SpatialReference()
        target.ImportFromEPSG(int(OutEPSG))
        transform = osr.CoordinateTransformation(source, target)

        Line = LineSegs('Path')
        try:
            with open(VUTProject, 'r', encoding='utf-8') as File:
                counter = 0
                i = 0
                PrevCourse = None
                PrevPos = None
                PrevHPr = None
                FirstPos = None
                FirstHpr = None

                for line in File:
                    if counter < 6:
                        counter += 1
                        continue
                    
                    elements = line.split()
                    if len(elements) < 7:
                        counter += 1
                        continue
                        
                    try:
                        lat = float(elements[0])
                        lon = float(elements[1])
                        ele = float(elements[2])
                        course = float(elements[4])
                        pitch = float(elements[5])
                        roll = float(elements[6])
                    except ValueError:
                        counter += 1
                        continue

                    if course < 180:
                        course = -course
                    elif course > 180:
                        course = abs(course - 360)

                    point = ogr.Geometry(ogr.wkbPoint)
                    point.AddPoint(lon, lat)
                    point.Transform(transform)
                    
                    x_coord = point.GetX() - Origin[0]
                    y_coord = point.GetY() - Origin[1]

                    if i == 0:
                        FirstPos = (x_coord, y_coord, ele)
                        FirstHpr = (course, pitch, roll)
                        self.cam.setPos(FirstPos)
                        self.cam.setHpr(FirstHpr)
                        Line.move_to(x_coord, y_coord, ele)
                    elif i == 1:
                        self.Moves.append(LerpPosHprInterval(self.cam, 1, (
                            x_coord, y_coord, ele), (fitDestAngle2Src(PrevCourse, course), pitch, roll),
                            startPos=FirstPos, startHpr=FirstHpr,
                            name='Interval', other=self.render))
                        Line.draw_to(x_coord, y_coord, ele)
                    else:
                        self.Moves.append(LerpPosHprInterval(self.cam, 1, (
                            x_coord, y_coord, ele), (fitDestAngle2Src(PrevCourse, course), pitch, roll),
                            startPos=PrevPos, startHpr=PrevHPr,
                            name='Interval', other=self.render))
                        Line.draw_to(x_coord, y_coord, ele)
                        
                    i += 1
                    PrevCourse = course
                    PrevPos = (x_coord, y_coord, ele)
                    PrevHPr = (course, pitch, roll)
                    counter += 1

            Line.setColor(1, 0.5, 0.5, 1)
            Line.setThickness(3)
            node = Line.create(False)
            self.render.attachNewNode(node)
            
        except Exception as e:
            logging.error(f"비행 프로젝트 파일 파싱 중 예외 발생: {e}")


class NetworkController:
    """안전한 패킷 파싱 레이어와 소켓 통신을 관리하는 클래스"""
    def __init__(self, taskMgr, data_callback):
        self.taskMgr = taskMgr
        self.data_callback = data_callback
        self.activeConnections = []
        self.cManager = None
        self.cListener = None
        self.cReader = None

    def setup_socket(self, port_address=9098, backlog=1000):
        self.cManager = QueuedConnectionManager()
        self.cListener = QueuedConnectionListener(self.cManager, 0)
        self.cReader = QueuedConnectionReader(self.cManager, 0)
        
        try:
            tcpSocket = self.cManager.openTCPServerRendezvous(port_address, backlog)
            self.cListener.addConnection(tcpSocket)
        except Exception as e:
            logging.error(f"통신 소켓 바인딩 실패 (포트 점유 가능성): {e}")
            return

        def tskListenerPolling(taskdata):
            if self.cListener.newConnectionAvailable():
                rendezvous = PointerToConnection()
                netAddress = NetAddress()
                newConnection = PointerToConnection()

                if self.cListener.getNewConnection(rendezvous, netAddress, newConnection):
                    conn = newConnection.p()
                    self.activeConnections.append(conn)
                    self.cReader.addConnection(conn)
            return Task.cont

        def tskReaderPolling(taskdata):
            if self.cReader.dataAvailable():
                datagram = NetDatagram()
                if self.cReader.getData(datagram):
                    self._safe_process_datagram(datagram)
            return Task.cont

        self.taskMgr.add(tskReaderPolling, "Poll the connection reader", -40)
        self.taskMgr.add(tskListenerPolling, "Poll the connection listener", -39)

    def _safe_process_datagram(self, netDatagram):
        """[네트워크 방어층] 예외 격리 및 데이터 무결성 검증"""
        try:
            myIterator = PyDatagramIterator(netDatagram)
            msgID = myIterator.getUint8()
            raw_str = myIterator.getString()
            messageToPrint = raw_str.split(',') if raw_str else []

            if msgID not in range(1, 6):
                logging.warning(f"알 수 없는 메시지 ID 무시됨: {msgID}")
                return

            if self.data_callback:
                self.data_callback(msgID, messageToPrint)
        except Exception:
            logging.exception("잘못된 네트워크 패킷 수신으로 인한 예외 무시됨")

    def shutdown(self):
        for conn in self.activeConnections:
            try:
                conn.close()
            except Exception:
                pass


class Video_UAV_Tracker_3D(ShowBase):

    def __init__(self, Input16bitTif, Texture, HFilmSize, VFilmSize, FocalLenght, VUTProject, Directory, videoFile, VideoWidth, VideoHeight, StartSecond, BBXMin, BBYMin, BBXMax, BBYMax):
        
        load_prc_file_data("", """
            textures-power-2 none
            gl-coordinate-system default
            window-title Video UAV Tracker 3D
        """)

        ShowBase.__init__(self)

        self.set_background_color(0.4, 0.4, 1)
        self.setFrameRateMeter(True)

        self.lens = PerspectiveLens()
        self.lens.setFilmSize(float(HFilmSize)/1000, float(VFilmSize)/1000)
        self.lens.setFocalLength(float(FocalLenght)/1000)
        base.cam.node().setLens(self.lens)

        self.Directory = Directory
        self.VideoFile = videoFile
        self.VideoWidth = int(VideoWidth)
        self.VideoHeight = int(VideoHeight)
        self.StartSecond = float(StartSecond)
        self.VRTBoundingBox = f"{BBXMin},{BBYMin}:{BBXMax},{BBYMax}"

        # 매니저 객체 초기화
        self.resource_manager = ResourceManager()
        atexit.register(self.resource_manager.cleanup)  # 프로세스 종료 시 자동 임시파일 소멸 보장

        self.terrain_manager = TerrainManager(self.resource_manager)
        self.path_manager = FlightPathManager(self.render, self.cam)
        self.network_controller = NetworkController(self.taskMgr, self.handle_network_message)

        try:
            self.terrain_manager.process_dem(Input16bitTif)
            self.terrain_manager.process_texture(Texture)
            self.SetupVisibleTerrain()
            self.SetupBulletTerrain()
        except Exception as e:
            logging.error(f"지형 및 텍스처 초기화 중 치명적 오류 발생: {e}")
            self.shutdown()
            sys.exit(1)

        self.accept("f3", self.toggleWireframe)
        
        self.path_manager.build_path(VUTProject, self.terrain_manager.OutEPSG, self.terrain_manager.Origin)
        self.Moves = self.path_manager.Moves

        self.OutputDir = os.path.join(os.path.dirname(self.VideoFile), f"{os.path.splitext(os.path.basename(self.VideoFile))[0]}_Mosaic")
        self.Mosaic = False
        self.MosaicCounter = 0
        
        self.taskMgr.setupTaskChain('MosaicChain', numThreads=1, tickClock=None,
                       threadPriority=None, frameBudget=None,
                       frameSync=None, timeslicePriority=None)
        
        self.network_controller.setup_socket()
        self.SendReadySignal(str(Directory))

    def shutdown(self):
        """[안전 종료] Panda3D 런타임 종료 전 자원 명시적 수거"""
        logging.info("엔진 종료 시퀀스 실행 중...")
        self.network_controller.shutdown()
        self.resource_manager.cleanup()
        self.userExit()

    def ActivateMosaics(self):
        if not self.Mosaic:
            self.Mosaic = True
            self.TaskCounter = 0
            self.LastProjectedPolygon = None
            if not os.path.exists(self.OutputDir):
                os.makedirs(self.OutputDir, exist_ok=True)
            self.task_mgr.add(self.ProcessFrustrum, 'CreateMosaic', taskChain='MosaicChain')
        else:
            self.Mosaic = False
            self.task_mgr.remove('CreateMosaic')

    def RayTrace(self, ScreenPoint):
        pFrom = Point3()
        pTo = Point3()
        self.cam.node().getLens().extrude(ScreenPoint, pFrom, pTo)

        pFrom = self.render.getRelativePoint(self.cam, pFrom)
        pTo = self.render.getRelativePoint(self.cam, pTo)
        result = self.world.rayTestClosest(pFrom, pTo)

        result2 = self.render.getRelativePoint(self.worldNP, result.getHitPos())
        return (result2[0] + self.terrain_manager.Origin[0], result2[1] + self.terrain_manager.Origin[1], result2[2])

    def ProcessFrustrum(self, task):
        VideoTime = self.Moves.get_t()
        UL = LPoint2f(-0.9, 0.9)
        MU = LPoint2f(0, 0.9)
        UR = LPoint2f(0.9, 0.9)
        MR = LPoint2f(0.9, 0)
        LR = LPoint2f(0.9, -0.9)
        MD = LPoint2f(0, -0.9)
        LL = LPoint2f(-0.9, -0.9)
        ML = LPoint2f(-0.9, 0)

        UL_XYZ = self.RayTrace(UL)
        UR_XYZ = self.RayTrace(UR)
        LR_XYZ = self.RayTrace(LR)
        LL_XYZ = self.RayTrace(LL)

        ring = ogr.Geometry(type=ogr.wkbLinearRing)
        ring.AddPoint(UL_XYZ[0], UL_XYZ[1], 0)
        ring.AddPoint(UR_XYZ[0], UR_XYZ[1], 0)
        ring.AddPoint(LR_XYZ[0], LR_XYZ[1], 0)
        ring.AddPoint(LL_XYZ[0], LL_XYZ[1], 0)
        ring.AddPoint(UL_XYZ[0], UL_XYZ[1], 0)

        poly = ogr.Geometry(type=ogr.wkbPolygon)
        poly.AddGeometry(ring)

        if self.TaskCounter == 0:
            self.LastProjectedPolygon = poly
            self.TaskCounter = 1
        else:
            Area1 = poly.GetArea()
            intersection = self.LastProjectedPolygon.Intersection(poly)
            if intersection and Area1 > 0:
                result = intersection.GetArea() / Area1
                if result < self.MosaicOverlap:
                    Center = self.RayTrace(LPoint2f(0, 0))
                    MU_XYZ = self.RayTrace(MU)
                    MR_XYZ = self.RayTrace(MR)
                    MD_XYZ = self.RayTrace(MD)
                    ML_XYZ = self.RayTrace(ML)
                    PointList = [UL_XYZ, MU_XYZ, UR_XYZ, MR_XYZ, LR_XYZ, MD_XYZ, LL_XYZ, ML_XYZ, Center]
                    OutputFile = os.path.join(self.OutputDir, f'Mosaic_{self.MosaicCounter}.bmp')
                    ScriptName = os.path.join(os.path.dirname(__file__), 'CreateMosaic.py')
                    
                    cmd = [
                        sys.executable, ScriptName,
                        str(self.VideoFile), OutputFile,
                        str(self.StartSecond), str(VideoTime),
                        str(self.VideoWidth), str(self.VideoHeight),
                        str(PointList), str(self.terrain_manager.OutEPSG),
                        str(self.BoundingBoxStr)
                    ]
                    
                    try:
                        # [장애 전파 차단] 외부 프로세스 데드락 방지를 위한 타임아웃(60초) 적용
                        subprocess.run(cmd, check=True, capture_output=True, text=True, timeout=60)
                        self.MosaicCounter += 1
                        self.LastProjectedPolygon = poly
                    except subprocess.TimeoutExpired:
                        logging.error("모자이크 생성 프로세스 타임아웃 발생 (60초 초과)")
                    except subprocess.CalledProcessError as e:
                        logging.error(f"모자이크 프로세스 실행 실패: {e.stderr}")

        return Task.cont

    def RunModel(self, start, Starttime):
        while time.time() < Starttime:
            time.sleep(0.001)
        self.Moves.start(startT=start)

    def StopModel(self, stop):
        start = max(0.0, stop - 0.001)
        self.Moves.start(startT=start, endT=stop)

    def SendReadySignal(self, Directory):
        ready_path = os.path.join(Directory, 'tmpConnection.txt')
        with open(ready_path, 'w', encoding='utf-8') as output:
            output.write('1')

    def handle_network_message(self, msgID, messageToPrint):
        try:
            if msgID == 1:
                Starttime = float(messageToPrint[0])
                start = float(messageToPrint[1])
                self.RunModel(start, Starttime)
            elif msgID == 2:
                Stoptime = float(messageToPrint[0])
                self.StopModel(Stoptime)
            elif msgID == 3:
                # [안전 종료 전환] sys.exit(0) 대신 엔진 shutdown 호출
                self.shutdown()
            elif msgID == 4:
                self.MosaicOverlap = float(messageToPrint[1])
                self.ActivateMosaics()
            elif msgID == 5:
                pos = float(messageToPrint[0])
                Pixelx = float(messageToPrint[1])
                Pixely = float(messageToPrint[2])
                self.get2DPoint(pos, Pixelx, Pixely)
        except (IndexError, ValueError) as e:
            logging.error(f"네트워크 메시지 파싱 인자 오류 (msgID: {msgID}): {e}")

    def get2DPoint(self, time_val, Pixelx, Pixely):
        start = max(0.0, time_val - 0.1)
        self.Moves.start(startT=start, endT=time_val)
        while self.Moves.getT() < time_val:
            time.sleep(0.001)

        ScreenPointx = Pixelx / float(self.VideoWidth) * 2 - 1
        ScreenPointy = 1 - (Pixely / float(self.VideoHeight) * 2)
        ScreenPointXY = LPoint2f(ScreenPointx, ScreenPointy)

        UTMPoint = self.RayTrace(ScreenPointXY)
        source = osr.SpatialReference()
        source.ImportFromEPSG(int(self.terrain_manager.OutEPSG))
        target = osr.SpatialReference()
        target.ImportFromEPSG(4326)
        transform = osr.CoordinateTransformation(source, target)
        
        Point = ogr.Geometry(ogr.wkbPoint)
        Point.AddPoint(UTMPoint[0], UTMPoint[1])
        Point.Transform(transform)
        
        coord_path = os.path.join(self.Directory, 'tmpCoordinate.txt')
        with open(coord_path, 'w', encoding='utf-8') as output:
            output.write(f"{Point.GetX()} {Point.GetY()} {UTMPoint[2]} {ScreenPointXY}\n")

    def SetupVisibleTerrain(self):
        tm = self.terrain_manager
        self.terrain_node = ShaderTerrainMesh()
        self.terrain_node.heightfield = self.loader.loadTexture(tm.PngDEM)
        self.terrain_node.target_triangle_width = 100.0
        self.terrain_node.generate()

        self.terrain = self.render.attach_new_node(self.terrain_node)
        self.terrain.set_scale(tm.PixelNr * tm.MeterScale, tm.PixelNr * tm.MeterScale, tm.HeightRange)

        terrain_shader = Shader.load(Shader.SL_GLSL, "terrain.vert.glsl", "terrain.frag.glsl")
        self.terrain.set_shader(terrain_shader)
        self.terrain.set_shader_input("camera", self.camera)

        grass_tex = self.loader.loadTexture(tm.TextureImage)
        grass_tex.set_anisotropic_degree(16)

        self.terrain.set_texture(grass_tex)
        self.terrain.setPos(0, 0, tm.OffsetHeight)

        # BoundingBox 문자열 계산 동기화
        parts = self.VRTBoundingBox.split(':')
        min_coords = parts[0].split(',')
        max_coords = parts[1].split(',')
        BBxMin, BByMin = float(min_coords[0]), float(min_coords[1])
        BBxMax, BByMax = float(max_coords[0]), float(max_coords[1])

        XLenght = BBxMax - BBxMin
        YLenght = BByMax - BByMin
        NewBBxMax = BBxMax + XLenght / 2
        NewBBxMin = BBxMin - XLenght / 2
        NewBByMax = BByMax + YLenght / 2
        NewBByMin = BByMin - YLenght / 2

        source = osr.SpatialReference()
        source.ImportFromEPSG(4326)
        target = osr.SpatialReference()
        target.ImportFromEPSG(int(tm.OutEPSG))
        transform = osr.CoordinateTransformation(source, target)

        pointMax = ogr.Geometry(ogr.wkbPoint)
        pointMax.AddPoint(NewBBxMax, NewBByMax)
        pointMax.Transform(transform)

        pointMin = ogr.Geometry(ogr.wkbPoint)
        pointMin.AddPoint(NewBBxMin, NewBByMin)
        pointMin.Transform(transform)

        self.BoundingBoxStr = f"-te {pointMin.GetX()} {pointMin.GetY()} {pointMax.GetX()} {pointMax.GetY()} "

    def SetupBulletTerrain(self):
        tm = self.terrain_manager
        self.worldNP = self.render.attachNewNode('World')
        self.world = BulletWorld()
        self.world.setGravity(Vec3(0, 0, -9.81))

        img = PNMImage(Filename(tm.PngDEM))
        shape = BulletHeightfieldShape(img, tm.HeightRange, ZUp)
        shape.setUseDiamondSubdivision(True)

        np = self.worldNP.attachNewNode(BulletRigidBodyNode('Heightfield'))
        np.node().addShape(shape)

        offset = tm.MeterScale * tm.PixelNr / 2.0
        np.setPos(+offset, +offset, +(tm.HeightRange / 2.0) + tm.OffsetHeight)

        np.setSx(tm.MeterScale)
        np.setSy(tm.MeterScale)
        np.setCollideMask(BitMask32.allOn())

        self.world.attachRigidBody(np.node())


if __name__ == '__main__':
    if len(sys.argv) < 16:
        print("Usage: python VUT_3D.py [Args 1 to 15...]")
        sys.exit(1)
        
    Video_UAV_Tracker_3D(
        sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[4], 
        sys.argv[5], sys.argv[6], sys.argv[7], sys.argv[8], 
        sys.argv[9], sys.argv[10], sys.argv[11], sys.argv[12], 
        sys.argv[13], sys.argv[14], sys.argv[15]
    ).run()

최종 개선사항
✅ 단일 스크립트 내 리소스 관리 → ResourceManager 분리 → 임시 파일 수명주기와 장애 종료 후 정리 안정성 확보
✅ GDAL/Panda3D 자원 직접 제어 → TerrainManager 추상화 → 지형 처리 로직 격리 및 메모리 누수 위험 감소
✅ 비행 데이터 파싱 로직 혼재 → FlightPathManager 분리 → GPS 경로 처리와 렌더링 책임 분리로 유지보수성 향상
✅ raw 네트워크 패킷 직접 처리 → NetworkController 방어 계층 추가 → 잘못된 입력으로 인한 엔진 크래시 방지
✅ os.system 기반 외부 실행 → subprocess 인자 배열 및 timeout 적용 → Command Injection 제거와 외부 프로세스 장애 전파 차단
✅ sys.exit 기반 강제 종료 → shutdown 생명주기 관리 전환 → 네트워크·파일·엔진 자원 정상 해제 보장
✅ 하드코딩된 처리 흐름 → Manager 기반 모듈 구조 전환 → 기능 확장 시 영향 범위 최소화 및 테스트 가능성 확보

원본은 연구용 단일 실행 스크립트 수준이었지만, 현재 버전은 보안·리소스·네트워크·프로세스 생명주기까지 방어층을 갖춘 운영 가능한 3D UAV 처리 엔진 구조로 승격되었다.    
