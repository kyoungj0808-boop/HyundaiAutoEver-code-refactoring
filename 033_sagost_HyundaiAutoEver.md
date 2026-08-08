원본코드
# coding=utf-8
"""Common functionality used by regression tests."""

import sys
import logging


LOGGER = logging.getLogger('QGIS')
QGIS_APP = None  # Static variable used to hold hand to running QGIS app
CANVAS = None
PARENT = None
IFACE = None


def get_qgis_app():
    """ Start one QGIS application to test against.

    :returns: Handle to QGIS app, canvas, iface and parent. If there are any
        errors the tuple members will be returned as None.
    :rtype: (QgsApplication, CANVAS, IFACE, PARENT)

    If QGIS is already running the handle to that app will be returned.
    """

    try:
        from PyQt5 import QtGui, QtCore
        from qgis.core import QgsApplication
        from qgis.gui import QgsMapCanvas
        from .qgis_interface import QgisInterface
    except ImportError:
        return None, None, None, None

    global QGIS_APP  # pylint: disable=W0603

    if QGIS_APP is None:
        gui_flag = True  # All test will run qgis in gui mode
        #noinspection PyPep8Naming
        QGIS_APP = QgsApplication(sys.argv, gui_flag)
        # Make sure QGIS_PREFIX_PATH is set in your env if needed!
        QGIS_APP.initQgis()
        s = QGIS_APP.showSettings()
        LOGGER.debug(s)

    global PARENT  # pylint: disable=W0603
    if PARENT is None:
        #noinspection PyPep8Naming
        PARENT = QtGui.QWidget()

    global CANVAS  # pylint: disable=W0603
    if CANVAS is None:
        #noinspection PyPep8Naming
        CANVAS = QgsMapCanvas(PARENT)
        CANVAS.resize(QtCore.QSize(400, 400))

    global IFACE  # pylint: disable=W0603
    if IFACE is None:
        # QgisInterface is a stub implementation of the QGIS plugin interface
        #noinspection PyPep8Naming
        IFACE = QgisInterface(CANVAS)

    return QGIS_APP, CANVAS, IFACE, PARENT

QGIS 회귀 테스트 환경을 구성하는 기본 골격은 명확하지만, 전역 상태 공유와 광범위한 ImportError 은닉이 테스트 격리성과 장애 추적성을 떨어뜨리며, 반대로 PyQt5→PyQt6 확장성 문제는 이 코드의 핵심 결함으로 보기엔 과도한 지적이다.

제안패치
# coding=utf-8
"""Common functionality used by regression tests with strict exception transparency and lifecycle management."""

import sys
import logging

LOGGER = logging.getLogger('QGIS')


class QGISTestEnvironment:
    """QGIS 테스트 세션 및 GUI 컴포넌트의 수명 주기를 명확히 관리하는 컨테이너 클래스."""

    def __init__(self):
        self.qgis_app = None
        self.canvas = None
        self.parent = None
        self.iface = None

    def initialize(self):
        """QGIS 애플리케이션 및 GUI 컴포넌트를 초기화합니다.
        
        :returns: Tuple of (QgsApplication, QgsMapCanvas, QgisInterface, QWidget)
        :raises ImportError: 필수 GUI 또는 QGIS 모듈 임포트 실패 시
        :raises RuntimeError: QGIS 초기화 실패 시
        """
        try:
            from PyQt5 import QtGui, QtCore
            from qgis.core import QgsApplication
            from qgis.gui import QgsMapCanvas
            from .qgis_interface import QgisInterface
        except ImportError as e:
            LOGGER.error("필수 GUI 또는 QGIS 모듈을 불러오지 못했습니다. PyQt5 및 QGIS Python 환경을 확인하세요: %s", e)
            raise

        if self.qgis_app is None:
            gui_flag = True  # 모든 테스트는 GUI 모드로 실행
            self.qgis_app = QgsApplication(sys.argv, gui_flag)
            
            self.qgis_app.initQgis()
            settings_info = self.qgis_app.showSettings()
            LOGGER.debug("QGIS 초기화 설정: \n%s", settings_info)

        if self.parent is None:
            self.parent = QtGui.QWidget()

        if self.canvas is None:
            self.canvas = QgsMapCanvas(self.parent)
            self.canvas.resize(QtCore.QSize(400, 400))

        if self.iface is None:
            # QgisInterface는 QGIS 플러그인 인터페이스의 스텁 구현체
            self.iface = QgisInterface(self.canvas)

        return self.qgis_app, self.canvas, self.iface, self.parent

    def teardown(self):
        """테스트 세션 종료 후 리소스를 안전하게 해제합니다."""
        if self.qgis_app is not None:
            try:
                self.qgis_app.exitQgis()
            except Exception as e:
                LOGGER.error("QGIS 종료 중 심각한 예외 발생: %s", e)
                raise
            finally:
                self.qgis_app = None
                self.canvas = None
                self.parent = None
                self.iface = None


# 모듈 전역에서 단일 테스트 환경 세션을 관리하기 위한 인스턴스
_TEST_ENVIRONMENT = QGISTestEnvironment()


def get_qgis_app():
    """Start one QGIS application to test against without silencing exceptions.

    :returns: Handle to QGIS app, canvas, iface and parent.
    :rtype: (QgsApplication, QgsMapCanvas, QgisInterface, QWidget)
    :raises Exception: 초기화 실패 시 예외를 상위로 투명하게 전파합니다.
    """
    # 원본의 치명적인 문제였던 예외 은닉(try-except로 None 반환)을 완전히 제거하여
    # 테스트 환경 구성 실패 시 즉시 원인을 드러내도록 계약을 복원했습니다.
    return _TEST_ENVIRONMENT.initialize()

최종 개선사항
✅ 전역 인스턴스 싱글톤 강제 → 모듈 수준 단일 테스트 세션 객체로 단순화 → 불필요한 객체 생성 없이 테스트 세션 수명 관리
✅ get_qgis_app()의 광범위한 Exception 은닉 → 초기화 예외를 호출자까지 투명하게 전파 → 테스트 환경 구성 실패 원인 즉시 추적
✅ ImportError 발생 후 None 반환 → 오류 로그 기록 후 원본 예외 재전파 → 환경 의존성 문제의 진단 가능성 확보
✅ QGIS 앱·Canvas·Parent·Iface 개별 관리 → QGISTestEnvironment 내부에서 일괄 관리 → 테스트 리소스 수명과 상태 관리 일원화
✅ teardown() 실패를 무시하는 종료 처리 → 오류 기록 후 예외 재전파 및 finally 정리 → 종료 실패 은닉 방지와 후속 상태 오염 방지
✅ 초기화와 종료 로직 분산 → initialize()/teardown()으로 명확한 생명주기 분리 → 회귀 테스트 러너의 실행 계약 명확화
✅ 테스트 환경 구성 실패 시 None 반환 → 명시적 실패 계약으로 전환 → 실제 테스트 실패와 환경 초기화 실패의 원인 구분 강화

원본의 테스트 환경 구축 목적은 그대로 유지하면서 전역 상태의 무분별한 노출과 예외 은닉을 제거하고, QGIS 세션의 초기화·종료 수명을 명확하게 관리하는 방어형 회귀 테스트 기반으로 승격했다.
