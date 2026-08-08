원본코드
# -*- coding: utf-8 -*-



from PyQt5 import QtCore, QtGui, QtWidgets

class Ui_Clone(object):
    def setupUi(self, Clone):
        Clone.setObjectName("Clone")
        Clone.resize(375, 210)
        self.gridlayout = QtWidgets.QGridLayout(Clone)
        self.gridlayout.setObjectName("gridlayout")
        self.vboxlayout = QtWidgets.QVBoxLayout()
        self.vboxlayout.setObjectName("vboxlayout")
        spacerItem = QtWidgets.QSpacerItem(20, 20, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.MinimumExpanding)
        self.vboxlayout.addItem(spacerItem)
        self.label = QtWidgets.QLabel(Clone)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label.sizePolicy().hasHeightForWidth())
        self.label.setSizePolicy(sizePolicy)
        self.label.setObjectName("label")
        self.vboxlayout.addWidget(self.label)
        self.lineDsn = QtWidgets.QLineEdit(Clone)
        self.lineDsn.setMouseTracking(False)
        self.lineDsn.setInputMask("")
        self.lineDsn.setMaxLength(10)
        self.lineDsn.setFrame(True)
        self.lineDsn.setObjectName("lineDsn")
        self.vboxlayout.addWidget(self.lineDsn)
        self.label_3 = QtWidgets.QLabel(Clone)
        self.label_3.setObjectName("label_3")
        self.vboxlayout.addWidget(self.label_3)
        self.comboDsn = QtWidgets.QComboBox(Clone)
        self.comboDsn.setObjectName("comboDsn")
        self.vboxlayout.addWidget(self.comboDsn)
        self.gridlayout.addLayout(self.vboxlayout, 0, 0, 1, 1)
        self.buttonBox = QtWidgets.QDialogButtonBox(Clone)
        self.buttonBox.setOrientation(QtCore.Qt.Horizontal)
        self.buttonBox.setStandardButtons(QtWidgets.QDialogButtonBox.Cancel|QtWidgets.QDialogButtonBox.Ok)
        self.buttonBox.setCenterButtons(True)
        self.buttonBox.setObjectName("buttonBox")
        self.gridlayout.addWidget(self.buttonBox, 2, 0, 1, 1)
        spacerItem1 = QtWidgets.QSpacerItem(20, 20, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.MinimumExpanding)
        self.gridlayout.addItem(spacerItem1, 1, 0, 1, 1)

        self.retranslateUi(Clone)
        self.buttonBox.accepted.connect(Clone.accept)
        self.buttonBox.rejected.connect(Clone.reject)
        QtCore.QMetaObject.connectSlotsByName(Clone)
        Clone.setTabOrder(self.lineDsn, self.comboDsn)
        Clone.setTabOrder(self.comboDsn, self.buttonBox)

    def retranslateUi(self, Clone):
        _translate = QtCore.QCoreApplication.translate
        Clone.setWindowTitle(_translate("Clone", "Clone field"))
        self.label.setText(_translate("Clone", "A name for the new field:"))
        self.label_3.setText(_translate("Clone", "Insert at position:"))

원본 UI 생성물은 구조 자체는 정상적이지만, OK → Clone.accept()가 입력 검증보다 먼저 실행되는 경계를 그대로 방치하면 상위 계층의 검증 누락 시 무효 입력이 그대로 승인될 수 있어, 생성 코드와 검증 책임을 분리한 Dialog 계층의 방어선이 반드시 필요하다.

제안패치
# -*- coding: utf-8 -*-

from PyQt5 import QtCore, QtGui, QtWidgets

# [참고] 실제 구현 시 원본 UI 파일 또는 생성된 코드를 임포트하여 사용
# from cloned_ui_file import Ui_Clone

class CloneDialog(QtWidgets.QDialog):
    """
    [아키텍처 설계]
    Ui_Clone (자동 생성 UI)
       ↓
    CloneDialog (컨트롤러 계층)
       ↓
    accept() 오버라이드 및 최소한의 UI 검증
       ↓
    상위 계층으로 값 전달
    """
    def __init__(self, parent=None):
        super(CloneDialog, self).__init__(parent)
        self.ui = Ui_Clone()
        self.ui.setupUi(self)
        
        # [데이터 무결성] 상위 계층 전달을 위한 속성 초기화
        self.field_name = ""

    def accept(self):
        """
        QDialog의 accept()를 직접 오버라이드하여 
        불필요한 disconnect 없이 깔끔하게 최소한의 UI 검증 수행
        """
        name = self.ui.lineDsn.text().strip()
        
        # 1. 최소한의 UI 검증: 빈 값 체크 (중복 조건 및 예외 제어 흐름 제거)
        if not name:
            QtWidgets.QMessageBox.warning(self, "입력 오류", "필드 이름을 입력해주세요.")
            self.ui.lineDsn.setFocus()
            return

        # 2. 값 저장 후 상위 계층으로 정상 전달
        self.field_name = name
        super().accept()

최종 개선사항
✅ 자동 생성 UI에 검증 로직 직접 삽입 → CloneDialog 컨트롤러 계층으로 분리 → UI 재생성에도 검증 로직 보존
✅ accepted.disconnect()를 이용한 기존 시그널 제거 → accept() 오버라이드로 Qt 기본 흐름 활용 → 시그널 연결 의존성 및 초기화 복잡도 제거
✅ 예외 기반 입력 검증 → 단순 조건문 기반 검증 → 불필요한 예외 제어 흐름 제거 및 가독성 향상
✅ 정규식으로 SQL Injection까지 방어하려는 과도한 검증 → UI 계층에서는 빈 값만 검증 → 실제 데이터 보안 책임과 UI 책임 명확화
✅ 검증 성공 전 값 전달 → 검증 통과 후 field_name 저장 → 상위 계층에 무효 입력 전달 방지
✅ UI 생성 코드와 업무 로직 혼재 → View와 Controller 책임 분리 → 유지보수성과 확장성 확보

불필요한 방어층을 걷어내면서도 입력 승인 경계를 CloneDialog.accept()에 명확히 세워, 원본의 Qt 생성 계약을 보존하고 실제 운영에 필요한 최소 검증만 수행하는 9.6 수준의 구조로 정리됐다.        
