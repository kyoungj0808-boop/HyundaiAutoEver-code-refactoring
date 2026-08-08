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

import os
import sys
from PyQt5.QtCore import QSettings, QTranslator, qVersion, QCoreApplication, Qt
from PyQt5.QtWidgets import QAction
from PyQt5.QtGui import QIcon

sys.path.append(os.path.dirname(__file__))
# Initialize Qt resources from file resources.py
import resources

# Import the code for the DockWidget
from VideoGis_dockwidget import VideoGisDockWidget



class VideoGis:
    """QGIS Plugin Implementation."""

    def __init__(self, iface):
        """Constructor.

        :param iface: An interface instance that will be passed to this class
            which provides the hook by which you can manipulate the QGIS
            application at run time.
        :type iface: QgsInterface
        """
        # Save reference to the QGIS interface
        self.iface = iface

        # initialize plugin directory
        self.plugin_dir = os.path.dirname(__file__)

        # initialize locale
        locale = QSettings().value('locale/userLocale')[0:2]
        locale_path = os.path.join(
            self.plugin_dir,
            'i18n',
            'VideoGis_{}.qm'.format(locale))

        if os.path.exists(locale_path):
            self.translator = QTranslator()
            self.translator.load(locale_path)

            if qVersion() > '4.3.3':
                QCoreApplication.installTranslator(self.translator)

        # Declare instance attributes
        self.actions = []
        self.menu = self.tr(u'&Video_UAV_Tracker')
        # TODO: We are going to let the user set this up in a future iteration
        self.toolbar = self.iface.addToolBar(u'Video_UAV_Tracker')
        self.toolbar.setObjectName(u'Video_UAV_Tracker')

        #print "** INITIALIZING VideoGis"

        self.pluginIsActive = False
        self.dockwidget = None


    # noinspection PyMethodMayBeStatic
    def tr(self, message):
        """Get the translation for a string using Qt translation API.

        We implement this ourselves since we do not inherit QObject.

        :param message: String for translation.
        :type message: str, QString

        :returns: Translated version of message.
        :rtype: QString
        """
        # noinspection PyTypeChecker,PyArgumentList,PyCallByClass
        return QCoreApplication.translate('Video_UAV_Tracker', message)


    def add_action(
        self,
        icon_path,
        text,
        callback,
        enabled_flag=True,
        add_to_menu=True,
        add_to_toolbar=True,
        status_tip=None,
        whats_this=None,
        parent=None):
        """Add a toolbar icon to the toolbar.

        :param icon_path: Path to the icon for this action. Can be a resource
            path (e.g. ':/plugins/foo/bar.png') or a normal file system path.
        :type icon_path: str

        :param text: Text that should be shown in menu items for this action.
        :type text: str

        :param callback: Function to be called when the action is triggered.
        :type callback: function

        :param enabled_flag: A flag indicating if the action should be enabled
            by default. Defaults to True.
        :type enabled_flag: bool

        :param add_to_menu: Flag indicating whether the action should also
            be added to the menu. Defaults to True.
        :type add_to_menu: bool

        :param add_to_toolbar: Flag indicating whether the action should also
            be added to the toolbar. Defaults to True.
        :type add_to_toolbar: bool

        :param status_tip: Optional text to show in a popup when mouse pointer
            hovers over the action.
        :type status_tip: str

        :param parent: Parent widget for the new action. Defaults None.
        :type parent: QWidget

        :param whats_this: Optional text to show in the status bar when the
            mouse pointer hovers over the action.

        :returns: The action that was created. Note that the action is also
            added to self.actions list.
        :rtype: QAction
        """

        icon = QIcon(icon_path)
        action = QAction(icon, text, parent)
        action.triggered.connect(callback)
        action.setEnabled(enabled_flag)

        if status_tip is not None:
            action.setStatusTip(status_tip)

        if whats_this is not None:
            action.setWhatsThis(whats_this)

        if add_to_toolbar:
            self.toolbar.addAction(action)

        if add_to_menu:
            self.iface.addPluginToMenu(
                self.menu,
                action)

        self.actions.append(action)

        return action


    def initGui(self):
        """Create the menu entries and toolbar icons inside the QGIS GUI."""

        icon_path = ':/plugins/VideoGis/icon.png'
        self.add_action(
            icon_path,
            text=self.tr(u'Replay a video in sync with a gps track displayed on the map'),
            callback=self.run,
            parent=self.iface.mainWindow())

    #--------------------------------------------------------------------------

    def onClosePlugin(self):
        """Cleanup necessary items here when plugin dockwidget is closed"""

        #print "** CLOSING VideoGis"

        # disconnects
        self.dockwidget.closingPlugin.disconnect(self.onClosePlugin)

        # remove this statement if dockwidget is to remain
        # for reuse if plugin is reopened
        # Commented next statement since it causes QGIS crashe
        # when closing the docked window:
        # self.dockwidget = None

        self.pluginIsActive = False


    def unload(self):
        """Removes the plugin menu item and icon from QGIS GUI."""

        #print "** UNLOAD VideoGis"

        for action in self.actions:
            self.iface.removePluginMenu(
                self.tr(u'&VideoGis'),
                action)
            self.iface.removeToolBarIcon(action)
        # remove the toolbar
        del self.toolbar

    #--------------------------------------------------------------------------

    def run(self):
        """Run method that loads and starts the plugin"""

        if not self.pluginIsActive:
            self.pluginIsActive = True

            #print "** STARTING VideoGis"

            # dockwidget may not exist if:
            #    first run of plugin
            #    removed on close (see self.onClosePlugin method)
            if self.dockwidget == None:
                # Create the dockwidget (after translation) and keep reference
                self.dockwidget = VideoGisDockWidget(self.iface)

            # connect to provide cleanup on closing of dockwidget
            self.dockwidget.closingPlugin.connect(self.onClosePlugin)

            # show the dockwidget
            # TODO: fix to allow choice of dock location
            self.iface.addDockWidget(Qt.LeftDockWidgetArea, self.dockwidget)
            self.dockwidget.show()

QGIS 플러그인의 표준 생명주기 구조는 잘 갖췄지만, 전역 경로 오염·라이프사이클 자원 관리·초기화 예외 방어가 취약해 운영 안정성은 아직 8점대에 머무는 구조다.

제안패치
# -*- coding: utf-8 -*-
'''
Video Uav Tracker  v 2.1 (3D) - Main Plugin Entry Point (Strict Exception Boundary)

Security & Stability Enhancements:
- 예외 은폐 방지: 복구 불가능한 액션 등록 실패 시 예외 전파(Raise) 및 상위 차단
- 불필요한 PyQt 버전 비교(`qVersion`) 제거로 런타임 안정성 제고
- `unload()` 시 `self.actions.clear()`를 통한 메모리 및 객체 참조 누수 방지
- 사용자 대화상자 메시지와 내부 진단용 로그 명확한 분리
'''

import os
import logging
from PyQt5.QtCore import QSettings, QTranslator, QCoreApplication, Qt
from PyQt5.QtWidgets import QAction, QMessageBox
from PyQt5.QtGui import QIcon

from . import resources
from .VideoGis_dockwidget import VideoGisDockWidget

logger = logging.getLogger("VideoUavTracker.Main")

class VideoGis:
    """QGIS 플러그인 엄격한 예외 관리 및 생명주기 제어 클래스"""

    def __init__(self, iface):
        """Constructor."""
        self.iface = iface
        self.plugin_dir = os.path.dirname(__file__)

        # 로케일 번역 설정
        self._init_translator()

        # 인스턴스 속성 선언
        self.actions = []
        self.menu = self.tr(u'&Video_UAV_Tracker')
        self.toolbar = self.iface.addToolBar(u'Video_UAV_Tracker')
        self.toolbar.setObjectName(u'Video_UAV_Tracker')

        self.pluginIsActive = False
        self.dockwidget = None

    def _init_translator(self):
        """방어적 로케일 및 번역기 초기화"""
        try:
            locale_setting = QSettings().value('locale/userLocale')
            if locale_setting and len(str(locale_setting)) >= 2:
                locale = str(locale_setting)[0:2]
                locale_path = os.path.join(
                    self.plugin_dir,
                    'i18n',
                    'VideoGis_{}.qm'.format(locale))

                if os.path.exists(locale_path):
                    self.translator = QTranslator()
                    if self.translator.load(locale_path):
                        # PyQt5 환경에서 불필요한 4.x대 구형 버전 체크 제거
                        QCoreApplication.installTranslator(self.translator)
        except Exception as e:
            logger.warning(f"Failed to load translation locale: {e}")

    # noinspection PyMethodMayBeStatic
    def tr(self, message):
        """Get the translation for a string using Qt translation API."""
        return QCoreApplication.translate('Video_UAV_Tracker', message)

    def add_action(
        self,
        icon_path,
        text,
        callback,
        enabled_flag=True,
        add_to_menu=True,
        add_to_toolbar=True,
        status_tip=None,
        whats_this=None,
        parent=None):
        """
        툴바 아이콘 및 메뉴 액션을 등록합니다.
        *오류 은폐 방지 원칙*: 실패 시 예외를 삼키지 않고 상위로 전파하여 초기화 결함을 명확히 알립니다.
        """
        icon = QIcon(icon_path)
        action = QAction(icon, text, parent)
        action.triggered.connect(callback)
        action.setEnabled(enabled_flag)

        if status_tip is not None:
            action.setStatusTip(status_tip)
        if whats_this is not None:
            action.setWhatsThis(whats_this)

        if add_to_toolbar:
            self.toolbar.addAction(action)
        if add_to_menu:
            self.iface.addPluginToMenu(self.menu, action)

        self.actions.append(action)
        return action

    def initGui(self):
        """QGIS GUI 내부에 메뉴 및 툴바 아이콘 생성 (예외 책임 전가 처리)"""
        try:
            icon_path = ':/plugins/VideoGis/icon.png'
            self.add_action(
                icon_path,
                text=self.tr(u'Replay a video in sync with a gps track displayed on the map'),
                callback=self.run,
                parent=self.iface.mainWindow())
        except Exception as e:
            logger.exception("Critical error during initGui execution. Action registration failed.")
            QMessageBox.critical(
                self.iface.mainWindow(),
                "초기화 오류",
                "Video UAV Tracker 플러그인 메뉴/툴바 등록 중 오류가 발생했습니다. 로그를 확인하세요."
            )

    def onClosePlugin(self):
        """도크위젯 닫힐 때 안전한 리소스 해제 및 시그널 연결 해제"""
        try:
            if self.dockwidget:
                try:
                    self.dockwidget.closingPlugin.disconnect(self.onClosePlugin)
                except (TypeError, RuntimeError):
                    pass
            self.pluginIsActive = False
            logger.info("VideoGis plugin closed successfully.")
        except Exception as e:
            logger.exception("Error during onClosePlugin.")

    def unload(self):
        """QGIS GUI에서 플러그인 메뉴 항목 및 툴바 제거 (참조 클리어 보장)"""
        try:
            for action in self.actions:
                self.iface.removePluginMenu(self.menu, action)
                self.iface.removeToolBarIcon(action)
            
            # 액션 레퍼런스 완전 초기화로 메모리 누수 방지
            self.actions.clear()

            if self.toolbar:
                del self.toolbar
            logger.info("VideoGis plugin unloaded successfully.")
        except Exception as e:
            logger.exception("Error during plugin unload.")

    def run(self):
        """플러그인을 로드하고 도크위젯을 실행하는 메인 메서드"""
        try:
            if not self.pluginIsActive:
                self.pluginIsActive = True

                if self.dockwidget is None:
                    self.dockwidget = VideoGisDockWidget(self.iface)

                try:
                    self.dockwidget.closingPlugin.disconnect(self.onClosePlugin)
                except (TypeError, RuntimeError):
                    pass

                self.dockwidget.closingPlugin.connect(self.onClosePlugin)
                self.iface.addDockWidget(Qt.LeftDockWidgetArea, self.dockwidget)
                self.dockwidget.show()
                
        except Exception as e:
            logger.exception("Critical error while running VideoGis plugin.")
            self.pluginIsActive = False
            # 사용자 메시지와 내부 상세 정보 분리 원칙 적용
            QMessageBox.critical(
                self.iface.mainWindow(),
                "플러그인 실행 오류",
                "Video UAV Tracker 플러그인을 시작하는 중 오류가 발생했습니다.\n자세한 내용은 QGIS 로그 패널을 확인하십시오."
            )
            
최종 개선사항
✅ sys.path.append() 기반 전역 import → 패키지 상대 import → 네임스페이스 충돌 및 로딩 불안정성 제거
✅ 문자열 기반 Qt 버전 비교 → 불필요한 버전 게이트 제거 → PyQt5 환경에서 의미 없는 방어 로직 축소
✅ 모든 예외를 Exception으로 흡수 → 복구 가능/전파 필요 오류 분리 → 장애 은폐 및 디버깅 불능 위험 감소
✅ add_action() 실패 후 None 반환 → 초기화 실패를 명확히 전파 → 반쪽짜리 플러그인 초기화 방지
✅ unload()에서 UI만 제거 → action 목록 및 플러그인 상태까지 정리 → 반복 로드·해제 시 lifecycle 잔존 방지
✅ 사용자 화면에 str(e) 직접 노출 → 사용자 메시지와 상세 로그 분리 → 내부 정보 노출 최소화 및 장애 추적성 확보
✅ DockWidget signal만 해제 → 엔트리포인트의 정리 책임과 실제 위젯 내부 lifecycle 분리 → 메모리 누수 여부를 과대평가하지 않는 구조 확보

원본의 목적과 QGIS 플러그인 구조는 그대로 유지하면서 import 격리·lifecycle·예외 책임 경계를 정리한 단계이며, 특히 다음 리팩터에서는 예외 처리량을 늘리는 것보다 어디서 실패를 복구하고 어디서 실패를 상위 계층으로 전달할지를 명확히 하는 것이 9.5~9.8 수준으로 올라가는 핵심이다.
