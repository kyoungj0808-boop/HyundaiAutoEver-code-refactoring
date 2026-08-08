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
from PyQt5 import QtGui, uic ,  QtWidgets
from PyQt5.QtCore import pyqtSignal
from PyQt5.QtWidgets import  QFileDialog
from NewProject import NewProject
from QGisMap import QGisMap
import resources

FORM_CLASS, _ = uic.loadUiType(os.path.join(
    os.path.dirname(__file__), 'VideoGis_dockwidget_base.ui'), resource_suffix='')


class VideoGisDockWidget(QtWidgets.QDockWidget, FORM_CLASS):

    closingPlugin = pyqtSignal()

    def __init__(self,iface, parent=None):
        """Constructor."""
        super(VideoGisDockWidget, self).__init__(parent)
        # Set up the user interface from Designer.
        # After setupUI you can access any designer object by doing
        # self.<objectname>, and you can use autoconnect slots - see
        # http://qt-project.org/doc/qt-4.8/designer-using-a-ui-file.html
        # #widgets-and-dialogs-with-auto-connect
        self.setupUi(self)
        self.iface = iface
        self.pushButton_2.setEnabled(False)
        self.lineEdit_2.setEnabled(False)
        self.pushButton_8.clicked.connect(self.close)
        self.pushButton_7.clicked.connect(self.NewProj)
        self.pushButton_5.clicked.connect(self.LoadProj)
        self.pushButton_2.clicked.connect(self.Start)

        self.QGisMapWindow = None
        self.NewProjectWindow = None
        self.groupBox.hide()
        
    def closeEvent(self, event):
        if self.QGisMapWindow != None:
            self.QGisMapWindow.close()
        if self.NewProjectWindow != None:
            self.NewProjectWindow.close()
        self.closingPlugin.emit()
        event.accept()

    def NewProj(self):
        self.projectfile = None
        self.pushButton_2.setEnabled(False)
        self.lineEdit_2.clear()
        projectfile, _ = QFileDialog.getSaveFileName(caption = 'Save project file', filter = "VUT Project (*.vut)")
        if projectfile:
            self.NewProjectWindow = NewProject(projectfile,self)
            #self.NewProjectWindow.setWindowModality(2)
            self.NewProjectWindow.show()
        
        
        
    def LoadProjFromNew(self,VutPrj):
        self.NewProjectWindow = None
        self.lineEdit_2.setText(VutPrj)  
        self.projectfile = VutPrj
        self.pushButton_2.setEnabled(True)
        
    
    def LoadProj(self):
        
        
        self.pushButton_2.setEnabled(False)
        self.lineEdit_2.clear()
        self.projectfile, _ = QFileDialog.getOpenFileName(caption = "Select project file",filter = "VUT Project  (*.vut)")
        if self.projectfile != '':
            with open(self.projectfile,'r') as File:
                for line in File:
                    if line[0:23] == 'Video start at msecond:':
                        self.lineEdit_2.setText(self.projectfile)
                        self.pushButton_2.setEnabled(True)
                        break
                
    
    def Start(self):
        
        
        self.QGisMapWindow = QGisMap(self.projectfile,self)
        #self.QGisMapWindow.setWindowModality(2)
        self.QGisMapWindow.show()

Qt Designer 기반 UI 흐름과 자식 창 종료 구조는 목적에 맞게 잡혀 있지만, 프로젝트 파일 판별의 매직넘버 의존과 파일 I/O 예외 무방비, None 비교 및 상태 초기화 부족 때문에 정상 흐름 밖의 입력에서 GUI 안정성이 급격히 떨어지는 레거시 도크 위젯 코드다.

제안패치
# coding=utf-8
"""Video Uav Tracker 3D v2.1 - Dock Widget with strict state commitment and zero dead-code."""

import os
import logging

from PyQt5 import uic, QtWidgets
from PyQt5.QtCore import pyqtSignal
from PyQt5.QtWidgets import QFileDialog

from NewProject import NewProject
from QGisMap import QGisMap

LOGGER = logging.getLogger('QGIS')

FORM_CLASS, _ = uic.loadUiType(os.path.join(
    os.path.dirname(__file__), 'VideoGis_dockwidget_base.ui'), resource_suffix='')


class VideoGisDockWidget(QtWidgets.QDockWidget, FORM_CLASS):
    """비디오 UAV 트래커의 메인 도크 위젯으로 검증 기반 상태 커밋과 안전한 수명 주기를 보장합니다."""

    closingPlugin = pyqtSignal()

    def __init__(self, iface, parent=None):
        """Constructor."""
        super(VideoGisDockWidget, self).__init__(parent)
        self.setupUi(self)
        self.iface = iface
        
        # 초기 UI 상태 제어
        self.pushButton_2.setEnabled(False)
        self.lineEdit_2.setEnabled(False)
        
        # 시그널 슬롯 연결
        self.pushButton_8.clicked.connect(self.close)
        self.pushButton_7.clicked.connect(self.NewProj)
        self.pushButton_5.clicked.connect(self.LoadProj)
        self.pushButton_2.clicked.connect(self.Start)

        self.QGisMapWindow = None
        self.NewProjectWindow = None
        self.projectfile = None
        self.groupBox.hide()
        
    def closeEvent(self, event):
        """위젯 종료 시 자식 창들을 안전하게 동반 종료합니다."""
        if self.QGisMapWindow is not None:
            self.QGisMapWindow.close()
        if self.NewProjectWindow is not None:
            self.NewProjectWindow.close()
        self.closingPlugin.emit()
        event.accept()

    def NewProj(self):
        """새로운 프로젝트 파일 생성 대화상자를 엽니다."""
        self.projectfile = None
        self.pushButton_2.setEnabled(False)
        self.lineEdit_2.clear()
        
        projectfile, _ = QFileDialog.getSaveFileName(
            caption='Save project file', 
            filter="VUT Project (*.vut)"
        )
        if projectfile:
            self.NewProjectWindow = NewProject(projectfile, self)
            self.NewProjectWindow.show()
        
    def LoadProjFromNew(self, VutPrj):
        """새 프로젝트 생성 후 콜백으로 경로를 로드합니다."""
        self.NewProjectWindow = None
        self.lineEdit_2.setText(VutPrj)  
        self.projectfile = VutPrj
        self.pushButton_2.setEnabled(True)
        
    def LoadProj(self):
        """기존 프로젝트 파일을 선택하고 검증 성공 시에만 내부 상태를 커밋합니다."""
        self.pushButton_2.setEnabled(False)
        self.lineEdit_2.clear()
        
        selected_path, _ = QFileDialog.getOpenFileName(
            caption="Select project file",
            filter="VUT Project  (*.vut)"
        )
        
        if selected_path:
            is_valid = False
            try:
                with open(selected_path, 'r', encoding='utf-8') as file_obj:
                    for line in file_obj:
                        # strip()을 추가하여 공백이나 개행 이슈를 원천 방어하고 검증 수행
                        if line.strip().startswith('Video start at msecond:'):
                            is_valid = True
                            break
            except (IOError, OSError, UnicodeDecodeError) as e:
                LOGGER.error("프로젝트 파일을 읽는 중 오류가 발생했습니다 (%s): %s", selected_path, e)
                QtWidgets.QMessageBox.critical(
                    self, 
                    "파일 오류", 
                    f"프로젝트 파일을 열 수 없습니다.\n상세 내용: {e}"
                )
                return

            # 검증이 완전히 성공했을 때만 내부 상태에 커밋하여 오염 방지
            if is_valid:
                self.projectfile = selected_path
                self.lineEdit_2.setText(self.projectfile)
                self.pushButton_2.setEnabled(True)
            else:
                LOGGER.warning("유효하지 않은 VUT 프로젝트 파일 형식입니다: %s", selected_path)
                QtWidgets.QMessageBox.warning(
                    self,
                    "형식 오류",
                    "선택한 파일은 유효한 VUT 프로젝트 파일이 아닙니다."
                )
    
    def Start(self):
        """검증된 프로젝트 설정으로 지도 및 트래커 창을 구동합니다."""
        if not self.projectfile:
            return
            
        self.QGisMapWindow = QGisMap(self.projectfile, self)
        self.QGisMapWindow.show()

최종 개선사항
✅ 파일 선택 즉시 내부 상태 변경 → 파일 내용 검증 후 상태 커밋 → 잘못된 프로젝트 선택에 따른 상태 오염 방지
✅ line[0:23] 매직넘버 파싱 → strip().startswith() 기반 명시적 검증 → 공백·개행 변화에 대한 파싱 견고성 확보
✅ 파일 I/O 예외 무방비 → OSError·UnicodeDecodeError를 명시적으로 처리 → 파일 접근·인코딩 장애의 사용자 피드백 및 원인 추적 확보
✅ 잘못된 파일 선택 시 불명확한 상태 → 검증 실패 시 버튼 비활성 상태 유지 → 실행 단계에서의 잘못된 프로젝트 진입 차단
✅ != None 기반 객체 검사 → is not None 적용 → 객체 존재 여부 판단의 명확성과 Python 관례 준수
✅ 사용하지 않는 sys·QtGui·resources 제거 → 실제 의존성만 유지 → 모듈 복잡도와 유지보수 비용 축소
✅ 초기화·로드·실행·종료 상태가 분산 관리 → projectfile 상태를 명시적으로 관리 → UI 상태와 실제 실행 가능 상태의 일관성 강화

원본의 프로젝트 생성·로드·실행·창 종료라는 기능 계약은 그대로 유지하면서, 검증되지 않은 파일이 내부 상태에 들어가는 경로를 차단하고 파일 장애를 명시적으로 처리하는 상태 커밋 구조로 승격했다.        
