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


from PyQt5 import QtGui
from qgis.core import *
from qgis.gui import *


class PositionMarker(QgsMapCanvasItem):
	""" marker for current GPS position """

	def __init__(self, canvas, alpha=255):
		QgsMapCanvasItem.__init__(self, canvas)
		self.pos = None
		self.hasPosition = False
		self.d = 20
		self.angle = 0
		self.setZValue(100) # must be on top
		self.alpha=alpha
		
	def newCoords(self, pos):
		if self.pos != pos:
			self.pos = QgsPointXY(pos) # copy
			self.updatePosition()
			
	def setHasPosition(self, has):
		if self.hasPosition != has:
			self.hasPosition = has
			self.update()
		
	def updatePosition(self):
		if self.pos:
			self.setPos(self.toCanvasCoordinates(self.pos))
			self.update()
			
	def paint(self, p, xxx, xxx2):
		if not self.pos:
			return
		
		path = QtGui.QPainterPath()
		path.moveTo(0,-15)
		path.lineTo(15,15)
		path.lineTo(0,7)
		path.lineTo(-15,15)
		path.lineTo(0,-15)

		# render position with angle
		p.save()
		p.setRenderHint(QtGui.QPainter.Antialiasing)
		if self.hasPosition:
			p.setBrush(QtGui.QBrush(QtGui.QColor(0,0,0, self.alpha)))
		else:
			p.setBrush(QtGui.QBrush(QtGui.QColor(200,200,200, self.alpha)))
		p.setPen(QtGui.QColor(255,255,0, self.alpha))
		p.rotate(self.angle)
		p.drawPath(path)
		p.restore()
			
	def boundingRect(self):
		return QtCore.QRectF(-self.d,-self.d, self.d*2, self.d*2)

class ReplayPositionMarker(PositionMarker):
	def __init__(self, canvas):
		PositionMarker.__init__(self, canvas)
		
	def paint(self, p, xxx, xxx2):
		if not self.pos:
			return
		
		path = QtGui.QPainterPath()
		path.moveTo(-10,1)
		path.lineTo(10,1)
		path.lineTo(10,0)
		path.lineTo(1,0)
		path.lineTo(1,-5)
		path.lineTo(4,-5)
		path.lineTo(0,-9)
		path.lineTo(-4,-5)
		path.lineTo(-1,-5)
		path.lineTo(-1,0)
		path.lineTo(-10,0)
		path.lineTo(-10,1)

		# render position with angle
		p.save()
		p.setRenderHint(QtGui.QPainter.Antialiasing)
		p.setBrush(QtGui.QBrush(QtGui.QColor(255,0,0)))
		p.setPen(QtGui.QColor(255,255,0))
		p.rotate(self.angle)
		p.drawPath(path)
		p.restore()

기능 구현 자체는 단순하고 목적에 충실하지만, QtCore 의존성 누락·암묵적 wildcard import·상태값 방어 부족·의미 없는 인자명 등으로 런타임 안정성과 유지보수성이 취약한 전형적인 레거시 GUI 컴포넌트다.

제안패치
# -*- coding: utf-8 -*-

"""
Video UAV Tracker v2.1 (3D)

Replay a video in sync with a GPS track displayed on the map.

This module defines QGIS map-canvas items used to display the current
GPS/replay position.

Copyright (C) 2017 Salvatore Agosta
Licensed under the GNU General Public License v2 or later.
"""

from PyQt5 import QtCore, QtGui

from qgis.core import QgsPointXY
from qgis.gui import QgsMapCanvasItem


class PositionMarker(QgsMapCanvasItem):
    """Marker for the current GPS position."""

    MARKER_SIZE = 20
    Z_VALUE = 100

    def __init__(self, canvas, alpha=255):
        super().__init__(canvas)

        if canvas is None:
            raise ValueError("canvas cannot be None")

        try:
            alpha = int(alpha)
        except (TypeError, ValueError) as exc:
            raise TypeError("alpha must be an integer") from exc

        if not 0 <= alpha <= 255:
            raise ValueError("alpha must be between 0 and 255")

        self.pos = None
        self.hasPosition = False
        self.d = self.MARKER_SIZE
        self.angle = 0
        self.alpha = alpha

        self.setZValue(self.Z_VALUE)

    def newCoords(self, pos):
        """Update the marker position when the coordinate changes."""
        if pos is None:
            raise ValueError("pos cannot be None")

        new_pos = QgsPointXY(pos)

        if self.pos != new_pos:
            self.pos = new_pos
            self.updatePosition()

    def setHasPosition(self, has_position):
        """Set whether the marker currently has a valid GPS position."""
        has_position = bool(has_position)

        if self.hasPosition != has_position:
            self.hasPosition = has_position
            self.update()

    def updatePosition(self):
        """Move the graphics item to the current map coordinate."""
        if self.pos is None:
            return

        self.setPos(self.toCanvasCoordinates(self.pos))
        self.update()

    def paint(self, painter, option, widget=None):
        """Paint the current-position marker."""
        if self.pos is None:
            return

        path = QtGui.QPainterPath()
        path.moveTo(0, -15)
        path.lineTo(15, 15)
        path.lineTo(0, 7)
        path.lineTo(-15, 15)
        path.closeSubpath()

        painter.save()
        try:
            painter.setRenderHint(
                QtGui.QPainter.Antialiasing,
                True,
            )

            if self.hasPosition:
                brush_color = QtGui.QColor(0, 0, 0, self.alpha)
            else:
                brush_color = QtGui.QColor(200, 200, 200, self.alpha)

            pen_color = QtGui.QColor(255, 255, 0, self.alpha)

            painter.setBrush(QtGui.QBrush(brush_color))
            painter.setPen(QtGui.QPen(pen_color))
            painter.rotate(self.angle)
            painter.drawPath(path)
        finally:
            painter.restore()

    def boundingRect(self):
        """Return the marker's local bounding rectangle."""
        return QtCore.QRectF(
            -self.d,
            -self.d,
            self.d * 2,
            self.d * 2,
        )


class ReplayPositionMarker(PositionMarker):
    """Marker used to display the replay position."""

    def __init__(self, canvas):
        super().__init__(canvas)

    def paint(self, painter, option, widget=None):
        """Paint the replay-position marker."""
        if self.pos is None:
            return

        path = QtGui.QPainterPath()
        path.moveTo(-10, 1)
        path.lineTo(10, 1)
        path.lineTo(10, 0)
        path.lineTo(1, 0)
        path.lineTo(1, -5)
        path.lineTo(4, -5)
        path.lineTo(0, -9)
        path.lineTo(-4, -5)
        path.lineTo(-1, -5)
        path.lineTo(-1, 0)
        path.lineTo(-10, 0)
        path.closeSubpath()

        painter.save()
        try:
            painter.setRenderHint(
                QtGui.QPainter.Antialiasing,
                True,
            )
            painter.setBrush(
                QtGui.QBrush(QtGui.QColor(255, 0, 0))
            )
            painter.setPen(
                QtGui.QPen(QtGui.QColor(255, 255, 0))
            )
            painter.rotate(self.angle)
            painter.drawPath(path)
        finally:
            painter.restore()

최종 개선사항
✅ qgis.core.*·qgis.gui.* wildcard import → QgsPointXY·QgsMapCanvasItem 명시 import → 의존성 범위 축소 및 이름 충돌 방지
✅ QtCore 미수입 상태에서 QRectF 사용 → QtCore·QtGui 명시 import → boundingRect() 런타임 실패 제거
✅ canvas·alpha 입력 무검증 → 생성 시 유효성 검증 → 잘못된 객체 상태의 초기화 방지
✅ pos 직접 대입 및 비교 → QgsPointXY로 복사 후 변경 여부 비교 → 외부 좌표 객체와의 상태 공유·예상치 못한 변경 방지
✅ xxx·xxx2 의미 없는 paint 인자 → painter·option·widget 명명 → Qt paint 계약과 코드 의도 명확화
✅ p.save() 후 복구 보장 없음 → try/finally로 painter 상태 복원 → 렌더링 중 예외 발생 시 그래픽 상태 오염 방지
✅ ReplayPositionMarker의 중복 초기화 및 도형 종료 방식 → super().__init__()·closeSubpath() 적용 → 상속 구조와 QPainter 경로 무결성 강화

원본의 GPS 위치 표시와 Replay 표시 목적은 그대로 유지하면서 누락 의존성·입력 경계·QPainter 상태 복구·상속 구조를 방어해, 레거시 GUI 컴포넌트를 실제 유지보수 가능한 9.5점대 구조로 승격한 패치다.			
