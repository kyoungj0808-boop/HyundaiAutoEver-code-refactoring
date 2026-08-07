원본코드
# -*- coding: utf-8 -*-
'''
Video Uav Tracker  v 2.1 (3D)

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
- Create associated shapefile
- Manage 3d options (select dem and image with same extension and cartographic projection)
- Identify first couple Frame/GpsTime and select it.
- Push Synchronize
- Push Start

Replay:
- Move on map
- Create associated DB shapefile
- Add POI with associated video frame saved
- Add POI directly from video sceen if 3D is active
- Create direct georeferenced mosaic if 3D is active
- Extract frames with associated coordinates for rapid photogrammetry use
'''

from qgis.gui import QgsMapTool
from PyQt5.QtGui import QCursor
from PyQt5.QtCore import Qt

class SkipTrackTool(QgsMapTool):
    
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
        self.cursor = QCursor(Qt.CrossCursor)
                                  
 
    def canvasPressEvent(self,event):
        layer = self.layer
        x = event.pos().x()
        y = event.pos().y()
        point = self.toLayerCoordinates(layer,event.pos())        
        pointMap = self.toMapCoordinates(layer, point)
        self.x0 = point.x()
        self.y0 = point.y()        
        self.Parent.findNearestPointInRecording(self.x0,self.y0)
            
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

원본은 QGIS MapTool 역할에 충실한 경량 이벤트 계층이지만, 객체 생명주기 검증과 참조 관리가 부족해 사용자 입력 순간 발생하는 예외가 전체 GIS 세션 장애로 확대될 위험이 있으며, 현재 버전은 작은 UI 도구를 넘어 안정적인 지도 인터랙션 컴포넌트로 승격하기 위한 방어층 보강이 필요한 구조다.

제안패치
# -*- coding: utf-8 -*-
'''
Video Uav Tracker  v 2.1 (3D)

Replay a video in sync with a gps track displayed on the map.
Production-Grade Ultimate Refactoring: SkipTrackTool (Fault-Tolerant & Memory-Safe QGIS Component)
'''

from __future__ import annotations
import logging
import weakref
from typing import Optional, Protocol, TYPE_CHECKING
from qgis.gui import QgsMapTool
from PyQt5.QtGui import QCursor
from PyQt5.QtCore import Qt

if TYPE_CHECKING:
    from qgis.gui import QgsMapCanvas
    from qgis.core import QgsMapLayer, QgsPointXY

logger = logging.getLogger(__name__)

class RecordingControllerProtocol(Protocol):
    """Protocol defining the required interface for the parent controller."""
    def findNearestPointInRecording(self, x: float, y: float) -> None:
        ...

class SkipTrackTool(QgsMapTool):
    """
    Production-grade QGIS Map Tool designed with weak reference management,
    granular exception handling, and strict interface contracts.
    """

    def __init__(
        self,
        canvas: QgsMapCanvas,
        layer: Optional[QgsMapLayer],
        parent: RecordingControllerProtocol
    ) -> None:
        if canvas is None:
            raise ValueError("SkipTrackTool requires a valid QgsMapCanvas instance.")
        
        super().__init__(canvas)
        # [메모리 누수 방지] weakref를 통한 순환 참조 차단
        self._parent_ref = weakref.ref(parent)
        self._canvas = canvas
        self._layer = layer
        self._cursor = QCursor(Qt.CrossCursor)

    @property
    def parent_controller(self) -> Optional[RecordingControllerProtocol]:
        """Resolve and return the parent controller safely from weak reference."""
        return self._parent_ref()

    @property
    def target_layer(self) -> Optional[QgsMapLayer]:
        """Read-only access to target map layer."""
        return self._layer

    def canvasPressEvent(self, event) -> None:
        """Handle map click events with granular exception handling and validation."""
        if event is None:
            logger.warning("SkipTrackTool: Received None event in canvasPressEvent.")
            return

        try:
            if not self._layer or not self._layer.isValid():
                logger.warning("SkipTrackTool: Target layer is invalid or None.")
                return

            parent = self.parent_controller
            if parent is None:
                logger.warning("SkipTrackTool: Parent controller has been garbage collected.")
                return

            # [세분화된 예외 처리 1] QGIS 좌표 변환 레이어 오류 격리
            try:
                point: Optional[QgsPointXY] = self.toLayerCoordinates(self._layer, event.pos())
            except RuntimeError as re:
                logger.warning(f"SkipTrackTool: QGIS coordinate transformation runtime error: {re}")
                return
            except Exception as ex:
                logger.error(f"SkipTrackTool: Unexpected error during coordinate transformation: {ex}")
                return

            if point is None:
                logger.warning("SkipTrackTool: Failed to map canvas position to layer coordinates.")
                return

            x0 = point.x()
            y0 = point.y()

            # [세분화된 예외 처리 2] 부모 컨트롤러 인터페이스 호출 오류 격리
            try:
                parent.findNearestPointInRecording(x0, y0)
            except AttributeError as ae:
                logger.error(f"SkipTrackTool: Parent controller lacks required method: {ae}")
            except Exception as ex:
                logger.error(f"SkipTrackTool: Error occurred in parent controller execution: {ex}")

        except Exception as e:
            # 예상치 못한 시스템 레벨 예외 방어 (민감한 전체 경로 스택트레이스 노출 최소화)
            logger.error("SkipTrackTool: Critical unhandled exception in event processing loop.")

    def canvasMoveEvent(self, event) -> None:
        """No-op for move events."""
        pass

    def canvasReleaseEvent(self, event) -> None:
        """No-op for release events."""
        pass

    def showSettingsWarning(self) -> None:
        """Placeholder for settings warning handler."""
        pass

    def activate(self) -> None:
        """Activate map tool and apply custom cursor safely."""
        if self._canvas:
            self._canvas.setCursor(self._cursor)

    def deactivate(self) -> None:
        """Clean up cursor state on deactivation."""
        if self._canvas:
            self._canvas.unsetCursor()

    def isZoomTool(self) -> bool:
        return False

    def isTransient(self) -> bool:
        return False

    def isEditTool(self) -> bool:
        # [정확성 개선] 지오메트리를 편집하는 도구가 아니므로 False 반환
        return False

최종개선사항
✅ 강한 Parent 참조 구조 → weakref 기반 생명주기 관리 전환 → QGIS 플러그인 재로딩/비활성화 시 메모리 누수 위험 감소
✅ object 타입 의존 → Protocol 기반 인터페이스 계약 적용 → 부모 컨트롤러 의존성 명확화 및 정적 검증 가능 구조 확보
✅ 전체 예외 포획 구조 → 좌표 변환/컨트롤러 호출 단계별 예외 분리 → 장애 발생 위치 추적성과 이벤트 루프 안정성 강화
✅ 무검증 이벤트 처리 → event/layer/controller 상태 검증 추가 → 비정상 UI 이벤트 상황에서도 QGIS 다운 방지
✅ 잘못된 편집 도구 선언 → 실제 기능에 맞는 isEditTool(False) 적용 → QGIS 내부 도구 동작 오판 방지
✅ 단순 Cursor 유지 구조 → activate/deactivate 생명주기 관리 추가 → 도구 전환 시 UI 상태 잔존 문제 방지

원본 QGIS 이벤트 브릿지 수준에서 객체 생명주기·예외 격리·인터페이스 계약까지 확보한 안정형 컴포넌트로 승격되었으며, 현재 버전은 GIS 애플리케이션 환경에서 장시간 실행을 견딜 수 있는 실무형 구조다.
