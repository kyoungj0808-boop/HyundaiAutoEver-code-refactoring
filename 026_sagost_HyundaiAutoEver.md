원본코드
# coding=utf-8
"""Safe Translations Test.

.. note:: This program is free software; you can redistribute it and/or modify
     it under the terms of the GNU General Public License as published by
     the Free Software Foundation; either version 2 of the License, or
     (at your option) any later version.

"""
from .utilities import get_qgis_app

__author__ = 'ismailsunni@yahoo.co.id'
__date__ = '12/10/2011'
__copyright__ = ('Copyright 2012, Australia Indonesia Facility for '
                 'Disaster Reduction')
import unittest
import os

from PyQt5.QtCore import QCoreApplication, QTranslator

QGIS_APP = get_qgis_app()


class SafeTranslationsTest(unittest.TestCase):
    """Test translations work."""

    def setUp(self):
        """Runs before each test."""
        if 'LANG' in iter(os.environ.keys()):
            os.environ.__delitem__('LANG')

    def tearDown(self):
        """Runs after each test."""
        if 'LANG' in iter(os.environ.keys()):
            os.environ.__delitem__('LANG')

    def test_qgis_translations(self):
        """Test that translations work."""
        parent_path = os.path.join(__file__, os.path.pardir, os.path.pardir)
        dir_path = os.path.abspath(parent_path)
        file_path = os.path.join(
            dir_path, 'i18n', 'af.qm')
        translator = QTranslator()
        translator.load(file_path)
        QCoreApplication.installTranslator(translator)

        expected_message = 'Goeie more'
        real_message = QCoreApplication.translate("@default", 'Good morning')
        self.assertEqual(real_message, expected_message)


if __name__ == "__main__":
    suite = unittest.makeSuite(SafeTranslationsTest)
    runner = unittest.TextTestRunner(verbosity=2)
    runner.run(suite)

원본은 실제 QGIS 번역 파이프라인을 검증하는 목적과 테스트 시나리오는 탄탄하지만, LANG 처리 중복과 전역 QCoreApplication 상태 오염, 레거시 makeSuite() 실행 구조가 남아 있어 테스트 격리성과 유지보수성을 떨어뜨리는 레거시 테스트 코드다.

제안패치
# coding=utf-8
"""Safe Translations Test.

.. note:: This program is free software; you can redistribute it and/or modify
     it under the terms of the GNU General Public License as published by
     the Free Software Foundation; either version 2 of the License, or
     (at your option) any later version.
"""

__author__ = 'ismailsunni@yahoo.co.id'
__date__ = '12/10/2011'
__copyright__ = ('Copyright 2012, Australia Indonesia Facility for '
                 'Disaster Reduction')

import os
import unittest
from pathlib import Path
from PyQt5.QtCore import QCoreApplication, QTranslator

from .utilities import get_qgis_app

# QGIS 애플리케이션 인스턴스 확보 (테스트 환경 의존성)
QGIS_APP = get_qgis_app()


class SafeTranslationsTest(unittest.TestCase):
    """Test translations work with clean state isolation and modern path handling."""

    def setUp(self) -> None:
        """Runs before each test: isolate environment and initialize state."""
        os.environ.pop('LANG', None)
        # 널(None) 대신 명시적인 인스턴스 속성 보장
        self.translator = QTranslator()

    def tearDown(self) -> None:
        """Runs after each test: clean up global translator state."""
        # 조건문 분기 없이 안전한 번역기 해제 및 환경 변수 정리
        QCoreApplication.removeTranslator(self.translator)
        os.environ.pop('LANG', None)

    def test_qgis_translations(self) -> None:
        """Test that QGIS translations work successfully."""
        # pathlib을 활용한 직관적이고 모던한 경로 계산
        file_path = Path(__file__).resolve().parents[1] / "i18n" / "af.qm"
        
        loaded = self.translator.load(str(file_path))
        self.assertTrue(loaded, f"Failed to load translation file from path: {file_path}")
        
        QCoreApplication.installTranslator(self.translator)

        expected_message = 'Goeie more'
        real_message = QCoreApplication.translate("@default", 'Good morning')
        self.assertEqual(
            real_message, 
            expected_message, 
            f"Translation mismatch. Expected '{expected_message}', but got '{real_message}'."
        )


if __name__ == "__main__":
    unittest.main(verbosity=2)

최종 개선사항
✅ LANG 존재 여부를 확인 후 삭제 → os.environ.pop('LANG', None)으로 단순화 → 테스트 환경 격리 및 중복 로직 제거
✅ Translator 미초기화 상태를 조건문으로 방어 → setUp()에서 QTranslator를 직접 생성 → 테스트 생명주기와 리소스 소유권 명확화
✅ 설치된 Translator를 테스트 종료 후 방치 → tearDown()에서 removeTranslator() 수행 → Qt 전역 번역 상태 오염 방지
✅ os.path 기반 다단계 경로 조합 → pathlib.Path 기반 경로 탐색 → 경로 처리 가독성과 유지보수성 강화
✅ 번역 파일 로드 결과 미검증 → load() 반환값을 즉시 assertion → 파일 누락과 번역 매핑 실패 원인 분리
✅ 번역 결과만 단순 비교 → 구체적인 실패 메시지 제공 → .qm 로딩 및 번역 불일치 원인 추적성 강화
✅ 수동 TestSuite/TextTestRunner 구성 → unittest.main(verbosity=2) 사용 → 테스트 실행 구조 단순화 및 자동 탐색 유지

원본의 실제 QGIS 번역 검증 목적은 그대로 보존하면서 환경 격리·Translator 수명·리소스 로딩 검증·경로 안정성을 정리해, 과설계 없이 운영 가능한 테스트 코드 수준으로 끌어올렸다.
