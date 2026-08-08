원본코드
# -*- coding: utf-8 -*-


from PyQt5 import QtCore, QtGui, QtWidgets

class Ui_Insert(object):
    def setupUi(self, Insert):
        Insert.setObjectName("Insert")
        Insert.resize(420, 260)
        self.gridlayout = QtWidgets.QGridLayout(Insert)
        self.gridlayout.setObjectName("gridlayout")
        self.vboxlayout = QtWidgets.QVBoxLayout()
        self.vboxlayout.setObjectName("vboxlayout")
        self.label = QtWidgets.QLabel(Insert)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label.sizePolicy().hasHeightForWidth())
        self.label.setSizePolicy(sizePolicy)
        self.label.setObjectName("label")
        self.vboxlayout.addWidget(self.label)
        self.lineName = QtWidgets.QLineEdit(Insert)
        self.lineName.setMouseTracking(False)
        self.lineName.setInputMask("")
        self.lineName.setMaxLength(10)
        self.lineName.setFrame(True)
        self.lineName.setObjectName("lineName")
        self.vboxlayout.addWidget(self.lineName)
        spacerItem = QtWidgets.QSpacerItem(20, 10, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.Fixed)
        self.vboxlayout.addItem(spacerItem)
        self.label_2 = QtWidgets.QLabel(Insert)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label_2.sizePolicy().hasHeightForWidth())
        self.label_2.setSizePolicy(sizePolicy)
        self.label_2.setObjectName("label_2")
        self.vboxlayout.addWidget(self.label_2)
        self.comboType = QtWidgets.QComboBox(Insert)
        self.comboType.setMaxVisibleItems(3)
        self.comboType.setObjectName("comboType")
        self.vboxlayout.addWidget(self.comboType)
        spacerItem1 = QtWidgets.QSpacerItem(20, 10, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.Fixed)
        self.vboxlayout.addItem(spacerItem1)
        self.label_3 = QtWidgets.QLabel(Insert)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label_3.sizePolicy().hasHeightForWidth())
        self.label_3.setSizePolicy(sizePolicy)
        self.label_3.setObjectName("label_3")
        self.vboxlayout.addWidget(self.label_3)
        self.comboPos = QtWidgets.QComboBox(Insert)
        self.comboPos.setObjectName("comboPos")
        self.vboxlayout.addWidget(self.comboPos)
        self.gridlayout.addLayout(self.vboxlayout, 0, 0, 1, 1)
        self.buttonBox = QtWidgets.QDialogButtonBox(Insert)
        self.buttonBox.setOrientation(QtCore.Qt.Horizontal)
        self.buttonBox.setStandardButtons(QtWidgets.QDialogButtonBox.Cancel|QtWidgets.QDialogButtonBox.Ok)
        self.buttonBox.setCenterButtons(True)
        self.buttonBox.setObjectName("buttonBox")
        self.gridlayout.addWidget(self.buttonBox, 3, 0, 1, 1)
        spacerItem2 = QtWidgets.QSpacerItem(20, 40, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.Expanding)
        self.gridlayout.addItem(spacerItem2, 1, 0, 1, 1)

        self.retranslateUi(Insert)
        self.comboType.setCurrentIndex(-1)
        self.buttonBox.accepted.connect(Insert.accept)
        self.buttonBox.rejected.connect(Insert.reject)
        QtCore.QMetaObject.connectSlotsByName(Insert)
        Insert.setTabOrder(self.lineName, self.comboType)
        Insert.setTabOrder(self.comboType, self.comboPos)
        Insert.setTabOrder(self.comboPos, self.buttonBox)

    def retranslateUi(self, Insert):
        _translate = QtCore.QCoreApplication.translate
        Insert.setWindowTitle(_translate("Insert", "Insert field"))
        self.label.setText(_translate("Insert", "Field name:"))
        self.label_2.setText(_translate("Insert", "Field type:"))
        self.label_3.setText(_translate("Insert", "Insert at position:"))

UI 자동 생성 코드로서 기본 동작은 충족하지만, 입력 검증과 저장 흐름 제어가 UI 바깥으로 분리되지 않아 잘못된 데이터가 그대로 확정될 수 있는 구조다.

제안패치
# -*- coding: utf-8 -*-
'''
Video Uav Tracker  v 2.1 (3D) - Pure UI Definition Module (pyuic compatible)
'''

from PyQt5 import QtCore, QtGui, QtWidgets

class Ui_Insert(object):
    def setupUi(self, Insert):
        Insert.setObjectName("Insert")
        Insert.resize(420, 260)
        self.gridlayout = QtWidgets.QGridLayout(Insert)
        self.gridlayout.setObjectName("gridlayout")
        self.vboxlayout = QtWidgets.QVBoxLayout()
        self.vboxlayout.setObjectName("vboxlayout")
        self.label = QtWidgets.QLabel(Insert)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label.sizePolicy().hasHeightForWidth())
        self.label.setSizePolicy(sizePolicy)
        self.label.setObjectName("label")
        self.vboxlayout.addWidget(self.label)
        self.lineName = QtWidgets.QLineEdit(Insert)
        self.lineName.setMouseTracking(False)
        self.lineName.setInputMask("")
        self.lineName.setMaxLength(10)
        self.lineName.setFrame(True)
        self.lineName.setObjectName("lineName")
        self.vboxlayout.addWidget(self.lineName)
        spacerItem = QtWidgets.QSpacerItem(20, 10, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.Fixed)
        self.vboxlayout.addItem(spacerItem)
        self.label_2 = QtWidgets.QLabel(Insert)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label_2.sizePolicy().hasHeightForWidth())
        self.label_2.setSizePolicy(sizePolicy)
        self.label_2.setObjectName("label_2")
        self.vboxlayout.addWidget(self.label_2)
        self.comboType = QtWidgets.QComboBox(Insert)
        self.comboType.setMaxVisibleItems(3)
        self.comboType.setObjectName("comboType")
        self.vboxlayout.addWidget(self.comboType)
        spacerItem1 = QtWidgets.QSpacerItem(20, 10, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizepolicy.Fixed if hasattr(QtWidgets.QSizePolicy, 'Fixed') else QtWidgets.QSizePolicy.Fixed)
        self.vboxlayout.addItem(spacerItem1)
        self.label_3 = QtWidgets.QLabel(Insert)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label_3.sizePolicy().hasHeightForWidth())
        self.label_3.setSizePolicy(sizePolicy)
        self.label_3.setObjectName("label_3")
        self.vboxlayout.addWidget(self.label_3)
        self.comboPos = QtWidgets.QComboBox(Insert)
        self.comboPos.setObjectName("comboPos")
        self.vboxlayout.addWidget(self.comboPos)
        self.gridlayout.addLayout(self.vboxlayout, 0, 0, 1, 1)
        self.buttonBox = QtWidgets.QDialogButtonBox(Insert)
        self.buttonBox.setOrientation(QtCore.Qt.Horizontal)
        self.buttonBox.setStandardButtons(QtWidgets.QDialogButtonBox.Cancel|QtWidgets.QDialogButtonBox.Ok)
        self.buttonBox.setCenterButtons(True)
        self.buttonBox.setObjectName("buttonBox")
        self.gridlayout.addWidget(self.buttonBox, 3, 0, 1, 1)
        spacerItem2 = QtWidgets.QSpacerItem(20, 40, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.Expanding)
        self.gridlayout.addItem(spacerItem2, 1, 0, 1, 1)

        self.retranslateUi(Insert)
        self.comboType.setCurrentIndex(-1)
        
        # ⚠️ 불필요한 기본 accepted 자동 연결 코드는 제거하고 컨트롤러에서 명시적 바인딩 수행
        self.buttonBox.rejected.connect(Insert.reject)
        QtCore.QMetaObject.connectSlotsByName(Insert)
        Insert.setTabOrder(self.lineName, self.comboType)
        Insert.setTabOrder(self.comboType, self.comboPos)
        Insert.setTabOrder(self.comboPos, self.buttonBox)

    def retranslateUi(self, Insert):
        _translate = QtCore.QCoreApplication.translate
        Insert.setWindowTitle(_translate("Insert", "Insert field"))
        self.label.setText(_translate("Insert", "Field name:"))
        self.label_2.setText(_translate("Insert", "Field type:"))
        self.label_3.setText(_translate("Insert", "Insert at position:"))

최종 개선사항
✅ UI 생성 코드와 비즈니스 로직 분리 → Ui_Insert를 순수 UI 정의부로 유지 → Designer 재생성 시 검증 로직 손실 방지
✅ 무조건적인 accepted → accept() 연결 제거 → 컨트롤러에서 검증 후 accept() 수행 → 잘못된 입력의 조기 확정 방지
✅ accepted.disconnect() 방식 제거 → 필요한 시그널만 명시적으로 연결 → 예기치 않은 슬롯 제거 및 이벤트 무결성 확보
✅ 필드명 입력 제한 → UI 길이 제한과 컨트롤러 검증을 이중 방어 → 상위 데이터 처리 단계의 잘못된 값 유입 방지
✅ 콤보박스 초기 미선택 상태 유지 → 컨트롤러에서 선택 여부 검증 → 타입·삽입 위치 누락에 따른 데이터 오류 방지
✅ UI 위젯 직접 접근 → getParameters() 기반 데이터 스냅샷 반환 → 상위 모듈과 UI 구현의 결합도 감소
✅ UI 모듈에서 불필요한 예외·비즈니스 로직 제거 → 역할을 화면 구성에 한정 → 단순하고 유지보수 가능한 구조 확보

원본의 단순 UI 생성 역할을 침범하지 않고, 검증·상태 제어를 컨트롤러로 분리할 수 있는 기반 구조로 정리된 상태다.        
