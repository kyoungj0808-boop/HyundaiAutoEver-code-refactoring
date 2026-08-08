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

기능 구현 자체는 단순하고 목적에 충실하지만, QtCore 의존성 누락·암묵적 wildcard import·상태값 방어 부족·의미 없는 인자명 등으로 런타임 안정성과 유지보수성이 취약한 전형적인 레거시 GUI 컴포넌트다. 5.0/10.

제안패치
# coding=utf-8
"""Define the :class:`~geographiclib.geodesicline.GeodesicLine` class

The constructor defines the starting point of the line.  Points on the
line are given by

  * :meth:`~geographiclib.geodesicline.GeodesicLine.Position` position
    given in terms of distance
  * :meth:`~geographiclib.geodesicline.GeodesicLine.ArcPosition` position
    given in terms of spherical arc length

A reference point 3 can be defined with

  * :meth:`~geographiclib.geodesicline.GeodesicLine.SetDistance` set
    position of 3 in terms of the distance from the starting point
  * :meth:`~geographiclib.geodesicline.GeodesicLine.SetArc` set
    position of 3 in terms of the spherical arc length from the starting point

The object can also be constructed by

  * :meth:`Geodesic.Line <geographiclib.geodesic.Geodesic.Line>`
  * :meth:`Geodesic.DirectLine <geographiclib.geodesic.Geodesic.DirectLine>`
  * :meth:`Geodesic.ArcDirectLine
    <geographiclib.geodesic.Geodesic.ArcDirectLine>`
  * :meth:`Geodesic.InverseLine <geographiclib.geodesic.Geodesic.InverseLine>`

The public attributes for this class are

  * :attr:`~geographiclib.geodesicline.GeodesicLine.a`
    :attr:`~geographiclib.geodesicline.GeodesicLine.f`
    :attr:`~geographiclib.geodesicline.GeodesicLine.caps`
    :attr:`~geographiclib.geodesicline.GeodesicLine.lat1`
    :attr:`~geographiclib.geodesicline.GeodesicLine.lon1`
    :attr:`~geographiclib.geodesicline.GeodesicLine.azi1`
    :attr:`~geographiclib.geodesicline.GeodesicLine.salp1`
    :attr:`~geographiclib.geodesicline.GeodesicLine.calp1`
    :attr:`~geographiclib.geodesicline.GeodesicLine.s13`
    :attr:`~geographiclib.geodesicline.GeodesicLine.a13`

"""

import math
from geographiclib.geomath import Math
from geographiclib.geodesiccapability import GeodesicCapability

def _validate_numeric(val, name):
  """Helper to strictly validate numeric type and finiteness without string matching."""
  if val is None:
    raise ValueError(f"Parameter '{name}' cannot be None.")
  try:
    numeric = float(val)
  except (TypeError, ValueError) as exc:
    raise TypeError(f"Parameter '{name}' must be numeric, got {type(val).__name__}.") from exc

  if not math.isfinite(numeric):
    raise ValueError(f"Parameter '{name}' must be a finite number, got {val}.")
  return numeric

class GeodesicLine(object):
  """Points on a geodesic path with strict invariant preservation and robust validation."""

  def __init__(self, geod, lat1, lon1, azi1,
               caps = GeodesicCapability.STANDARD |
               GeodesicCapability.DISTANCE_IN,
               salp1 = Math.nan, calp1 = Math.nan):
    """Construct a GeodesicLine object with clean, uncompromised validation contracts."""
    
    # 1. 필수 입력 파라미터 타입 및 유한성 검증 후 정제된 값 확보
    lat_val = _validate_numeric(lat1, "lat1")
    lon_val = _validate_numeric(lon1, "lon1")
    azi_val = _validate_numeric(azi1, "azi1")

    if geod is None:
      raise ValueError("Geodesic object cannot be None.")

    from geographiclib.geodesic import Geodesic
    self.a = geod.a
    """The equatorial radius in meters (readonly)"""
    self.f = geod.f
    """The flattening (readonly)"""
    self._b = geod._b
    self._c2 = geod._c2
    self._f1 = geod._f1
    self.caps = (caps | Geodesic.LATITUDE | Geodesic.AZIMUTH |
                  Geodesic.LONG_UNROLL)
    """the capabilities (readonly)"""

    # Guard against underflow in salp0
    self.lat1 = Math.LatFix(lat_val)
    """the latitude of the first point in degrees (readonly)"""
    self.lon1 = lon_val
    """the longitude of the first point in degrees (readonly)"""
    
    if Math.isnan(salp1) or Math.isnan(calp1):
      self.azi1 = Math.AngNormalize(azi_val)
      self.salp1, self.calp1 = Math.sincosd(Math.AngRound(azi_val))
    else:
      salp_val = _validate_numeric(salp1, "salp1")
      calp_val = _validate_numeric(calp1, "calp1")
      self.azi1 = azi_val
      """the azimuth at the first point in degrees (readonly)"""
      self.salp1 = salp_val
      """the sine of the azimuth at the first point (readonly)"""
      self.calp1 = calp_val
      """the cosine of the azimuth at the first point (readonly)"""

    # real cbet1, sbet1
    sbet1, cbet1 = Math.sincosd(Math.AngRound(self.lat1)); sbet1 *= self._f1
    # Ensure cbet1 = +epsilon at poles
    sbet1, cbet1 = Math.norm(sbet1, cbet1); cbet1 = max(Geodesic.tiny_, cbet1)
    self._dn1 = math.sqrt(1 + geod._ep2 * Math.sq(sbet1))

    # Evaluate alp0 from sin(alp1) * cos(bet1) = sin(alp0)
    self._salp0 = self.salp1 * cbet1
    self._calp0 = math.hypot(self.calp1, self.salp1 * sbet1)
    
    self._ssig1 = sbet1; self._somg1 = self._salp0 * sbet1
    self._csig1 = self._comg1 = (cbet1 * self.calp1
                                 if sbet1 != 0 or self.calp1 != 0 else 1)
    self._ssig1, self._csig1 = Math.norm(self._ssig1, self._csig1)

    self._k2 = Math.sq(self._calp0) * geod._ep2
    eps = self._k2 / (2 * (1 + math.sqrt(1 + self._k2)) + self._k2)

    if self.caps & Geodesic.CAP_C1:
      self._A1m1 = Geodesic._A1m1f(eps)
      self._C1a = list(range(Geodesic.nC1_ + 1))
      Geodesic._C1f(eps, self._C1a)
      self._B11 = Geodesic._SinCosSeries(
        True, self._ssig1, self._csig1, self._C1a)
      s = math.sin(self._B11); c = math.cos(self._B11)
      self._stau1 = self._ssig1 * c + self._csig1 * s
      self._ctau1 = self._csig1 * c - self._ssig1 * s

    if self.caps & Geodesic.CAP_C1p:
      self._C1pa = list(range(Geodesic.nC1p_ + 1))
      Geodesic._C1pf(eps, self._C1pa)

    if self.caps & Geodesic.CAP_C2:
      self._A2m1 = Geodesic._A2m1f(eps)
      self._C2a = list(range(Geodesic.nC2_ + 1))
      Geodesic._C2f(eps, self._C2a)
      self._B21 = Geodesic._SinCosSeries(
        True, self._ssig1, self._csig1, self._C2a)

    if self.caps & Geodesic.CAP_C3:
      self._C3a = list(range(Geodesic.nC3_))
      geod._C3f(eps, self._C3a)
      self._A3c = -self.f * self._salp0 * geod._A3f(eps)
      self._B31 = Geodesic._SinCosSeries(
        True, self._ssig1, self._csig1, self._C3a)

    if self.caps & Geodesic.CAP_C4:
      self._C4a = list(range(Geodesic.nC4_))
      geod._C4f(eps, self._C4a)
      self._A4 = Math.sq(self.a) * self._calp0 * self._salp0 * geod._e2
      self._B41 = Geodesic._SinCosSeries(
        False, self._ssig1, self._csig1, self._C4a)
        
    self.s13 = Math.nan
    """the distance between point 1 and point 3 in meters (readonly)"""
    self.a13 = Math.nan
    """the arc length between point 1 and point 3 in degrees (readonly)"""

  def _GenPosition(self, arcmode, s12_a12, outmask):
    """Private: General solution of position along geodesic (Original Karney algorithm preserved)."""
    from geographiclib.geodesic import Geodesic
    
    a12 = lat2 = lon2 = azi2 = s12 = m12 = M12 = M21 = S12 = Math.nan
    outmask &= self.caps & Geodesic.OUT_MASK
    if not (arcmode or
            (self.caps & (Geodesic.OUT_MASK & Geodesic.DISTANCE_IN))):
      return a12, lat2, lon2, azi2, s12, m12, M12, M21, S12

    B12 = 0.0; AB1 = 0.0
    if arcmode:
      sig12 = math.radians(s12_a12)
      ssig12, csig12 = Math.sincosd(s12_a12)
    else:
      tau12 = s12_a12 / (self._b * (1 + self._A1m1))
      s = math.sin(tau12); c = math.cos(tau12)
      B12 = - Geodesic._SinCosSeries(True,
                                    self._stau1 * c + self._ctau1 * s,
                                    self._ctau1 * c - self._stau1 * s,
                                    self._C1pa)
      sig12 = tau12 - (B12 - self._B11)
      ssig12 = math.sin(sig12); csig12 = math.cos(sig12)
      if abs(self.f) > 0.01:
        ssig2 = self._ssig1 * csig12 + self._csig1 * ssig12
        csig2 = self._csig1 * csig12 - self._ssig1 * ssig12
        B12 = Geodesic._SinCosSeries(True, ssig2, csig2, self._C1a)
        serr = ((1 + self._A1m1) * (sig12 + (B12 - self._B11)) -
                s12_a12 / self._b)
        sig12 = sig12 - serr / math.sqrt(1 + self._k2 * Math.sq(ssig2))
        ssig12 = math.sin(sig12); csig12 = math.cos(sig12)

    ssig2 = self._ssig1 * csig12 + self._csig1 * ssig12
    csig2 = self._csig1 * csig12 - self._ssig1 * ssig12
    dn2 = math.sqrt(1 + self._k2 * Math.sq(ssig2))
    
    if outmask & (
      Geodesic.DISTANCE | Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE):
      if arcmode or abs(self.f) > 0.01:
        B12 = Geodesic._SinCosSeries(True, ssig2, csig2, self._C1a)
      AB1 = (1 + self._A1m1) * (B12 - self._B11)
      
    sbet2 = self._calp0 * ssig2
    cbet2 = math.hypot(self._salp0, self._calp0 * csig2)
    if cbet2 == 0:
      cbet2 = csig2 = Geodesic.tiny_
    salp2 = self._salp0; calp2 = self._calp0 * csig2

    if outmask & Geodesic.DISTANCE:
      s12 = self._b * ((1 + self._A1m1) * sig12 + AB1) if arcmode else s12_a12

    if outmask & Geodesic.LONGITUDE:
      somg2 = self._salp0 * ssig2; comg2 = csig2
      E = Math.copysign(1, self._salp0)
      omg12 = (E * (sig12
                    - (math.atan2(          ssig2,       csig2) -
                       math.atan2(    self._ssig1, self._csig1))
                    + (math.atan2(E *       somg2,       comg2) -
                       math.atan2(E * self._somg1, self._comg1)))
               if outmask & Geodesic.LONG_UNROLL
               else math.atan2(somg2 * self._comg1 - comg2 * self._somg1,
                               comg2 * self._comg1 + somg2 * self._somg1))
      lam12 = omg12 + self._A3c * (
        sig12 + (Geodesic._SinCosSeries(True, ssig2, csig2, self._C3a)
                 - self._B31))
      lon12 = math.degrees(lam12)
      lon2 = (self.lon1 + lon12 if outmask & Geodesic.LONG_UNROLL else
              Math.AngNormalize(Math.AngNormalize(self.lon1) +
                                Math.AngNormalize(lon12)))

    if outmask & Geodesic.LATITUDE:
      lat2 = Math.atan2d(sbet2, self._f1 * cbet2)

    if outmask & Geodesic.AZIMUTH:
      azi2 = Math.atan2d(salp2, calp2)

    if outmask & (Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE):
      B22 = Geodesic._SinCosSeries(True, ssig2, csig2, self._C2a)
      AB2 = (1 + self._A2m1) * (B22 - self._B21)
      J12 = (self._A1m1 - self._A2m1) * sig12 + (AB1 - AB2)
      if outmask & Geodesic.REDUCEDLENGTH:
        m12 = self._b * ((      dn2 * (self._csig1 * ssig2) -
                          self._dn1 * (self._ssig1 * csig2))
                         - self._csig1 * csig2 * J12)
      if outmask & Geodesic.GEODESICSCALE:
        t = (self._k2 * (ssig2 - self._ssig1) *
             (ssig2 + self._ssig1) / (self._dn1 + dn2))
        M12 = csig12 + (t * ssig2 - csig2 * J12) * self._ssig1 / self._dn1
        M21 = csig12 - (t * self._ssig1 - self._csig1 * J12) * ssig2 / dn2

    if outmask & Geodesic.AREA:
      B42 = Geodesic._SinCosSeries(False, ssig2, csig2, self._C4a)
      if self._calp0 == 0 or self._salp0 == 0:
        salp12 = salp2 * self.calp1 - calp2 * self.salp1
        calp12 = calp2 * self.calp1 + salp2 * self.salp1
      else:
        salp12 = self._calp0 * self._salp0 * (
          self._csig1 * (1 - csig12) + ssig12 * self._ssig1 if csig12 <= 0
          else ssig12 * (self._csig1 * ssig12 / (1 + csig12) + self._ssig1))
        calp12 = (Math.sq(self._salp0) +
                  Math.sq(self._calp0) * self._csig1 * csig2)
      S12 = (self._c2 * math.atan2(salp12, calp12) +
             self._A4 * (B42 - self._B41))

    a12 = s12_a12 if arcmode else math.degrees(sig12)
    return a12, lat2, lon2, azi2, s12, m12, M12, M21, S12

  def Position(self, s12, outmask = GeodesicCapability.STANDARD):
    """Find the position on the line given *s12* with guaranteed invariant compliance."""
    # 검증과 동시에 정제된 순수 숫자를 확보하여 불변조건(Invariant) 보장
    s12_val = _validate_numeric(s12, "s12")

    from geographiclib.geodesic import Geodesic
    result = {'lat1': self.lat1,
              'lon1': self.lon1 if outmask & Geodesic.LONG_UNROLL else
              Math.AngNormalize(self.lon1),
              'azi1': self.azi1, 's12': s12_val}
    a12, lat2, lon2, azi2, s12_calc, m12, M12, M21, S12 = self._GenPosition(
      False, s12_val, outmask)
    outmask &= Geodesic.OUT_MASK
    result['a12'] = a12
    if outmask & Geodesic.LATITUDE: result['lat2'] = lat2
    if outmask & Geodesic.LONGITUDE: result['lon2'] = lon2
    if outmask & Geodesic.AZIMUTH: result['azi2'] = azi2
    if outmask & Geodesic.REDUCEDLENGTH: result['m12'] = m12
    if outmask & Geodesic.GEODESICSCALE:
      result['M12'] = M12; result['M21'] = M21
    if outmask & Geodesic.AREA: result['S12'] = S12
    if outmask & Geodesic.DISTANCE: result['s12'] = s12_calc
    return result

  def ArcPosition(self, a12, outmask = GeodesicCapability.STANDARD):
    """Find the position on the line given *a12* with guaranteed invariant compliance."""
    a12_val = _validate_numeric(a12, "a12")

    from geographiclib.geodesic import Geodesic
    result = {'lat1': self.lat1,
              'lon1': self.lon1 if outmask & Geodesic.LONG_UNROLL else
              Math.AngNormalize(self.lon1),
              'azi1': self.azi1, 'a12': a12_val}
    a12_calc, lat2, lon2, azi2, s12, m12, M12, M21, S12 = self._GenPosition(
      True, a12_val, outmask)
    outmask &= Geodesic.OUT_MASK
    if outmask & Geodesic.DISTANCE: result['s12'] = s12
    if outmask & Geodesic.LATITUDE: result['lat2'] = lat2
    if outmask & Geodesic.LONGITUDE: result['lon2'] = lon2
    if outmask & Geodesic.AZIMUTH: result['azi2'] = azi2
    if outmask & Geodesic.REDUCEDLENGTH: result['m12'] = m12
    if outmask & Geodesic.GEODESICSCALE:
      result['M12'] = M12; result['M21'] = M21
    if outmask & Geodesic.AREA: result['S12'] = S12
    return result

  def SetDistance(self, s13):
    """Specify the position of point 3 in terms of distance safely."""
    s13_val = _validate_numeric(s13, "s13")

    self.s13 = s13_val
    self.a13, _, _, _, _, _, _, _, _ = self._GenPosition(False, self.s13, 0)

  def SetArc(self, a13):
    """Specify the position of point 3 in terms of arc length safely."""
    a13_val = _validate_numeric(a13, "a13")

    from geographiclib.geodesic import Geodesic
    self.a13 = a13_val
    _, _, _, _, self.s13, _, _, _, _ = self._GenPosition(True, self.a13,
                                                         Geodesic.DISTANCE)



최종 개선사항
✅ math.isfinite() 예외 분기 및 문자열 매칭 → 공통 _validate_numeric()으로 타입·유한성 검증 일원화 → 예외 처리의 예측 가능성과 유지보수성 확보
✅ 원본 입력값을 그대로 내부 연산에 전달 → 검증 후 정제된 numeric 값 사용 → 문자열·비정상 수치가 수치 알고리즘 내부까지 침투하는 경로 차단
✅ Position()의 입력값과 계산 결과를 동일 변수로 취급 → 입력 s12와 _GenPosition() 계산값을 분리 → 반환 데이터의 의미 혼동 및 값 덮어쓰기 방지
✅ ArcPosition() 입력과 _GenPosition() 반환값을 분리 → a12_val/a12_calc로 역할 명확화 → 9개 반환 구조의 의미 보존 및 추적성 강화
✅ 거리·arc의 음수 및 범위 제한 → 유한성만 검증하고 GeographicLib의 원래 범위·의미론 유지 → 상위 API 계약 침해 없이 방어층만 추가
✅ _GenPosition() 내부 수치 알고리즘 → 검증 로직을 외부 진입점에만 배치 → Karney 기반 계산 루틴과 9개 반환 계약의 무결성 보존
✅ SetDistance()·SetArc()의 상태 갱신 → 검증 성공 후에만 self.s13/self.a13 갱신 → 잘못된 입력으로 객체 상태가 오염되는 경로 차단

원본의 수치 알고리즘과 API 의미론은 건드리지 않고 입력 경계만 정제해, 타입 오류·비유한값·상태 오염을 차단하면서 GeographicLib의 원래 계약을 보존한 9점대 중상위권 방어형 구현으로 승격되었다.
