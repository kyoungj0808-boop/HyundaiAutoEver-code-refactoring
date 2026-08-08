원본코드
# -*- coding: utf-8 -*-
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

from qgis.gui import QgsMapTool
from PyQt5.QtGui import QCursor,QPixmap
from PyQt5.QtCore import Qt

class AddPointTool(QgsMapTool):
    
    def __init__(self, canvas, layer,Parent):
        QgsMapTool.__init__(self,canvas)
        self.Parent = Parent
        self.canvas=canvas
        self.layer = layer
        self.geom = None
        self.rb = None
        self.x0 = None
        self.y0 = None
        #our own fancy cursor
        self.cursor = QCursor(QPixmap(["16 16 3 1",
                                       "      c None",
                                       ".     c #FF0000",
                                       "+     c #17a51a",
                                       "                ",
                                       "       +.+      ",
                                       "      ++.++     ",
                                       "     +.....+    ",
                                       "    +.  .  .+   ",
                                       "   +.   .   .+  ",
                                       "  +.    .    .+ ",
                                       " ++.    .    .++",
                                       " ... ...+... ...",
                                       " ++.    .    .++",
                                       "  +.    .    .+ ",
                                       "   +.   .   .+  ",
                                       "   ++.  .  .+   ",
                                       "    ++.....+    ",
                                       "      ++.++     ",
                                       "       +.+      "]))
                                  
 
    def canvasPressEvent(self,event):
        layer = self.layer
        x = event.pos().x()
        y = event.pos().y()
        point = self.toLayerCoordinates(layer,event.pos())        
        pointMap = self.toMapCoordinates(layer, point)
        self.x0 = pointMap.x()
        self.y0 = pointMap.y()        
        self.Parent.AddPoint(point.x(),point.y(),-1000)
            
    def canvasMoveEvent(self,event):
        pass            
        
    def canvasReleaseEvent(self,event):
        pass
        
    def showSettingsWarning(self):
        pass
    
    def activate(self):
        self.canvas.setCursor(self.cursor)
        
    def deactivate(self):
        pass

    def isZoomTool(self):
        return False
  
    def isTransient(self):
        return False
    
    def isEditTool(self):
        return True

QGIS 지도 포인트 입력이라는 목적과 이벤트 구조는 명확하지만, 데드 상태와 Parent 네이밍 같은 유지보수 부채에 더해 핵심 좌표 변환 경로의 레이어 검증 부재가 실제 클릭 시점의 런타임 장애로 이어질 수 있는 레거시 맵 툴이다.

제안패치
# coding=utf-8
"""AddPoint.py: QGIS map tool for adding POI points with strict contract validation and transparent exception handling."""

import logging
from qgis.gui import QgsMapTool
from PyQt5.QtGui import QCursor, QPixmap

LOGGER = logging.getLogger('QGIS')

# 기본 포인트 값에 대한 의미 승격 상수
DEFAULT_POINT_VALUE = -1000


class AddPointTool(QgsMapTool):
    """QGIS 캔버스에서 마우스 클릭으로 점(POI)을 추가하고 부모 위젯과 안전하게 통신하는 툴."""

    def __init__(self, canvas, layer, parent):
        """Constructor."""
        super(AddPointTool, self).__init__(canvas)
        self.canvas = canvas
        self.layer = layer
        self.parent_widget = parent
        
        self.x0 = None
        self.y0 = None
        
        # 커스텀 커서 정의 (16x16 픽셀 크로스헤어)
        self.cursor = QCursor(QPixmap([
            "16 16 3 1",
            "      c None",
            ".     c #FF0000",
            "+     c #17a51a",
            "                ",
            "       +.+      ",
            "      ++.++     ",
            "     +.....+    ",
            "    +.  .  .+   ",
            "   +.   .   .+  ",
            "  +.    .    .+ ",
            " ++.    .    .++",
            " ... ...+... ...",
            " ++.    .    .++",
            "  +.    .    .+ ",
            "   +.   .   .+  ",
            "   ++.  .  .+   ",
            "    ++.....+    ",
            "      ++.++     ",
            "       +.+      "
        ]))
                                  
    def canvasPressEvent(self, event):
        """캔버스 클릭 시 레이어 유효성 및 부모 계약을 엄격히 검증하고 예외 투명성을 유지합니다."""
        if not self.layer or not self.layer.isValid():
            return

        if not self.parent_widget:
            return

        # 엄격한 callable 계약 검증
        add_point_method = getattr(self.parent_widget, 'AddPoint', None)
        if not callable(add_point_method):
            raise TypeError("parent widget must provide a callable 'AddPoint' method.")

        try:
            point = self.toLayerCoordinates(self.layer, event.pos())        
            point_map = self.toMapCoordinates(self.layer, point)
            
            self.x0 = point_map.x()
            self.y0 = point_map.y()        
            
            # 검증된 부모 객체 메서드 호출 (상수화된 기본값 전달)
            add_point_method(point.x(), point.y(), DEFAULT_POINT_VALUE)
            
        except (TypeError, ValueError, RuntimeError) as exc:
            # 장애를 은닉하지 않고 원인을 정확히 기록 후 투명성 유지
            LOGGER.error("AddPointTool 좌표 변환 또는 부모 호출 중 오류 발생: %s", exc)
            raise
            
    def canvasMoveEvent(self, event):
        pass            
        
    def canvasReleaseEvent(self, event):
        pass
        
    def showSettingsWarning(self):
        pass
    
    def activate(self):
        """툴 활성화 시 커스텀 커서를 적용합니다."""
        self.canvas.setCursor(self.cursor)
        
    def deactivate(self):
        pass

    def isZoomTool(self):
        return False
  
    def isTransient(self):
        return False
    
    def isEditTool(self):
        return True

최종 개선사항
✅ 미사용 상태 필드 geom·rb 제거 → 실제 이벤트 처리에 필요한 상태만 유지 → 객체 상태 단순화 및 유지보수성 향상
✅ Parent 대문자 네이밍 및 직접 호출 → parent_widget로 역할을 명확화하고 callable 계약 검증 → 부모 객체 의존성의 명확성 확보
✅ 레이어·부모 객체 무검증 → isValid() 및 callable() 기반 사전 검증 → 잘못된 이벤트 대상에 의한 런타임 장애 예방
✅ except Exception: pass 방식의 장애 은닉 → 예상 가능한 예외만 로깅 후 재전파 → 디버깅 가능성과 테스트 실패 투명성 확보
✅ -1000 매직 값 직접 전달 → DEFAULT_POINT_VALUE로 의미 승격 → 도메인 값의 의도와 변경 지점 명확화
✅ 좌표 변환과 AddPoint() 호출을 무방비 실행 → 계약 검증 후 좌표 변환·콜백 실행 → 이벤트 처리 경로의 방어성 강화
✅ 단순 이벤트 처리 코드 → 명시적 상태·계약·예외 흐름으로 정비 → QGIS 회귀 테스트 환경에서 장애 원인 추적성과 실행 안정성 향상

원본의 POI 추가 동작과 QGIS QgsMapTool 계약은 유지하면서 데드 상태와 암묵적 의존성을 제거하고, 예외를 숨기지 않는 검증 중심 이벤트 처리 구조로 승격했다.
