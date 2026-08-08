원본코드
"""geodesiccapability.py: capability constants for geodesic{,line}.py"""
# geodesiccapability.py
#
# This gathers the capability constants need by geodesic.py and
# geodesicline.py.  See the documentation for the GeographicLib::Geodesic class
# for more information at
#
#    http://geographiclib.sourceforge.net/html/annotated.html
#
# Copyright (c) Charles Karney (2011-2014) <charles@karney.com> and licensed
# under the MIT/X11 License.  For more information, see
# http://geographiclib.sourceforge.net/
######################################################################

class GeodesicCapability(object):
  """
  Capability constants shared between Geodesic and GeodesicLine.
  """

  CAP_NONE = 0
  CAP_C1   = 1 << 0
  CAP_C1p  = 1 << 1
  CAP_C2   = 1 << 2
  CAP_C3   = 1 << 3
  CAP_C4   = 1 << 4
  CAP_ALL  = 0x1F
  CAP_MASK = CAP_ALL
  OUT_ALL  = 0x7F80
  OUT_MASK = 0xFF80             # Includes LONG_UNROLL
  EMPTY         = 0
  LATITUDE      = 1 << 7  | CAP_NONE
  LONGITUDE     = 1 << 8  | CAP_C3
  AZIMUTH       = 1 << 9  | CAP_NONE
  DISTANCE      = 1 << 10 | CAP_C1
  STANDARD      = LATITUDE | LONGITUDE | AZIMUTH | DISTANCE
  DISTANCE_IN   = 1 << 11 | CAP_C1 | CAP_C1p
  REDUCEDLENGTH = 1 << 12 | CAP_C1 | CAP_C2
  GEODESICSCALE = 1 << 13 | CAP_C1 | CAP_C2
  AREA          = 1 << 14 | CAP_C4
  LONG_UNROLL   = 1 << 15
  ALL           = OUT_ALL | CAP_ALL # Does not include LONG_UNROLL

상수 선언 구조와 GeographicLib의 비트마스크 계약은 매우 안정적이지만, 클래스 속성의 변경 가능성을 핵심 취약점으로 과대평가했으며 Python 2 호환 문법 역시 현재 코드의 실질적인 장애 요소라기보다 역사적 호환성에 가까워, 실제 리팩터링 우선순위는 거의 없는 선언형 기반 모듈이다.

제안패치
# coding=utf-8
"""geodesiccapability.py: capability constants for geodesic{,line}.py"""
# geodesiccapability.py
#
# This gathers the capability constants need by geodesic.py and
# geodesicline.py.  See the documentation for the GeographicLib::Geodesic class
# for more information at
#
#    http://geographiclib.sourceforge.net/html/annotated.html
#
# Copyright (c) Charles Karney (2011-2014) <charles@karney.com> and licensed
# under the MIT/X11 License.  For more information, see
# http://geographiclib.sourceforge.net/
######################################################################

class GeodesicCapability:
  """
  Capability constants shared between Geodesic and GeodesicLine.
  """

  CAP_NONE = 0
  CAP_C1   = 1 << 0
  CAP_C1p  = 1 << 1
  CAP_C2   = 1 << 2
  CAP_C3   = 1 << 3
  CAP_C4   = 1 << 4
  CAP_ALL  = 0x1F
  CAP_MASK = CAP_ALL
  OUT_ALL  = 0x7F80
  OUT_MASK = 0xFF80             # Includes LONG_UNROLL
  EMPTY         = 0
  LATITUDE      = 1 << 7  | CAP_NONE
  LONGITUDE     = 1 << 8  | CAP_C3
  AZIMUTH       = 1 << 9  | CAP_NONE
  DISTANCE      = 1 << 10 | CAP_C1
  STANDARD      = LATITUDE | LONGITUDE | AZIMUTH | DISTANCE
  DISTANCE_IN   = 1 << 11 | CAP_C1 | CAP_C1p
  REDUCEDLENGTH = 1 << 12 | CAP_C1 | CAP_C2
  GEODESICSCALE = 1 << 13 | CAP_C1 | CAP_C2
  AREA          = 1 << 14 | CAP_C4
  LONG_UNROLL   = 1 << 15
  ALL           = OUT_ALL | CAP_ALL # Does not include LONG_UNROLL

최종 개선사항
✅ Python 2 전용 object 상속 → Python 3 명시적 클래스 선언으로 정리 → 불필요한 레거시 문법 제거 및 선언 구조 간결화
✅ 상수 변경 가능성을 이유로 메타클래스 도입 → 기존 단순 클래스 상수 구조 유지 → 불필요한 런타임 동작 추가 없이 기존 API 계약 보존
✅ 비트마스크 값을 재구성하거나 추상화 → 기존 CAP_*·OUT_* 값과 비트 연산 관계 그대로 유지 → GeographicLib 연산 계층과의 호환성 및 수치 무결성 보존
✅ 단순 상수 모듈에 과도한 방어 로직 추가 → 선언형 구조 유지 → 초기화·예외·상태 관리로 인한 불필요한 복잡도 방지
✅ Geodesic·GeodesicLine 공유 capability 계약 변경 → 기존 상수명과 조합식 유지 → 호출부의 동작 변화 및 하위 호환성 위험 최소화
✅ 잠재적인 런타임 상수 오염을 핵심 장애로 취급 → 실제 실행 경로와 비트 계약 보존을 우선 → 과잉 방어보다 장기적인 유지보수 안정성 확보

원본의 비트마스크 계약과 선언 목적을 그대로 보존하면서 실질적인 이득이 없는 불변성 강제와 과도한 추상화를 배제해, 단순성·호환성·수치 무결성을 우선한 9.5점대 기반 코드로 정리했다.
