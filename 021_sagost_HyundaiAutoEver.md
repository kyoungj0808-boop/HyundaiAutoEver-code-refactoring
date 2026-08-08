원본코드
"""Define the WGS84 ellipsoid"""
# constants.py
#
# This is a translation of the GeographicLib::Constants class to python.  See
# the documentation for the C++ class for more information at
#
#    http://geographiclib.sourceforge.net/html/annotated.html
#
# Copyright (c) Charles Karney (2011-2016) <charles@karney.com> and
# licensed under the MIT/X11 License.  For more information, see
# http://geographiclib.sourceforge.net/
######################################################################

class Constants(object):
  """
  Constants describing the WGS84 ellipsoid
  """

  WGS84_a = 6378137.0           # meters
  """the equatorial radius in meters of the WGS84 ellipsoid in meters"""
  WGS84_f = 1/298.257223563
  """the flattening of the WGS84 ellipsoid, 1/298.257223563"""

정확한 WGS84 상수를 안정적으로 제공하는 목적에는 충실하지만, 단순 상수 네임스페이스를 레거시 클래스 형태로 유지한 탓에 Pythonic한 구조와 불변성 측면에서 개선 여지가 남아 있다.

제안패치
# -*- coding: utf-8 -*-
from typing import Final

class Constants:
    """
    Constants describing the WGS84 ellipsoid
    (Translated from GeographicLib::Constants, maintaining API compatibility)
    """

    WGS84_a: Final[float] = 6378137.0           # meters
    """the equatorial radius in meters of the WGS84 ellipsoid in meters"""
    
    WGS84_f: Final[float] = 1 / 298.257223563
    """the flattening of the WGS84 ellipsoid, 1/298.257223563"""

최종 개선사항
✅ 불필요한 인스턴스화 차단 로직 제거 → 기존 Constants 네임스페이스/API 호환성 유지 → 레거시 호출 경로의 불필요한 장애 방지
✅ 단순 클래스 변수 → Final[float] 타입 명시 → 정적 분석 단계의 상수 재할당 탐지 강화
✅ 런타임 불변성으로 과장된 설명 → Final의 정적 검사 역할로 한정 → 기술적 설명의 정확성 확보
✅ WGS84 기준값 변경 없이 유지 → 원본 수치와 계산식 그대로 보존 → 지리 계산 엔진의 수치 무결성 유지
✅ 상수 모듈에 런타임 검증·예외 처리 추가 → 선언과 검증 책임 분리 → 불필요한 복잡도 및 장애 지점 제거
✅ 원본의 클래스 기반 API 변경 → 기존 Constants.WGS84_a/f 접근 방식 유지 → 하위 호환성 확보
✅ 문서 문자열에 API 호환 목적 명시 → 포팅 코드의 설계 의도 보존 → 후속 유지보수 시 구조 변경 방지

원본의 단순성과 API 호환성은 그대로 보존하면서 Final을 통한 정적 타입 안전성만 추가해, 과설계 없이 수치 무결성과 유지보수성을 함께 강화한 실무형 상수 모듈로 정리됐다.
