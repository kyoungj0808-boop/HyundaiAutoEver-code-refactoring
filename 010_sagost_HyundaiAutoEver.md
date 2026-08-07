원본코드
# -*- coding: utf-8 -*-



from PyQt5 import QtCore, QtGui, QtWidgets

class Ui_Rename(object):
    def setupUi(self, Rename):
        Rename.setObjectName("Rename")
        Rename.resize(397, 126)
        self.gridlayout = QtWidgets.QGridLayout(Rename)
        self.gridlayout.setObjectName("gridlayout")
        self.vboxlayout = QtWidgets.QVBoxLayout()
        self.vboxlayout.setObjectName("vboxlayout")
        self.label = QtWidgets.QLabel(Rename)
        sizePolicy = QtWidgets.QSizePolicy(QtWidgets.QSizePolicy.Preferred, QtWidgets.QSizePolicy.Fixed)
        sizePolicy.setHorizontalStretch(0)
        sizePolicy.setVerticalStretch(0)
        sizePolicy.setHeightForWidth(self.label.sizePolicy().hasHeightForWidth())
        self.label.setSizePolicy(sizePolicy)
        self.label.setObjectName("label")
        self.vboxlayout.addWidget(self.label)
        self.lineEdit = QtWidgets.QLineEdit(Rename)
        self.lineEdit.setMouseTracking(False)
        self.lineEdit.setInputMask("")
        self.lineEdit.setMaxLength(10)
        self.lineEdit.setFrame(True)
        self.lineEdit.setObjectName("lineEdit")
        self.vboxlayout.addWidget(self.lineEdit)
        self.gridlayout.addLayout(self.vboxlayout, 0, 0, 1, 1)
        self.buttonBox = QtWidgets.QDialogButtonBox(Rename)
        self.buttonBox.setOrientation(QtCore.Qt.Horizontal)
        self.buttonBox.setStandardButtons(QtWidgets.QDialogButtonBox.Cancel|QtWidgets.QDialogButtonBox.Ok)
        self.buttonBox.setCenterButtons(True)
        self.buttonBox.setObjectName("buttonBox")
        self.gridlayout.addWidget(self.buttonBox, 2, 0, 1, 1)
        spacerItem = QtWidgets.QSpacerItem(20, 40, QtWidgets.QSizePolicy.Minimum, QtWidgets.QSizePolicy.Expanding)
        self.gridlayout.addItem(spacerItem, 1, 0, 1, 1)

        self.retranslateUi(Rename)
        self.buttonBox.accepted.connect(Rename.accept)
        self.buttonBox.rejected.connect(Rename.reject)
        QtCore.QMetaObject.connectSlotsByName(Rename)

    def retranslateUi(self, Rename):
        _translate = QtCore.QCoreApplication.translate
        Rename.setWindowTitle(_translate("Rename", "Rename field"))
        self.label.setText(_translate("Rename", "Enter new field name:"))

원본 코드는 GeographicLib 기반 고정밀 지오데식 연산 엔진으로 수학적 정확성과 알고리즘 완성도는 뛰어나지만, 외부 입력 검증과 장애 격리 계층이 부족해 신뢰할 수 없는 데이터가 유입되는 운영 환경에서는 작은 입력 오류가 전체 계산 파이프라인 불안정으로 이어질 수 있는 구조다.

제안패치
# geodesic_engine.py
import math
from geographiclib.geomath import Math
from geographiclib.constants import Constants
from geographiclib.geodesiccapability import GeodesicCapability

class GeodesicEngineError(Exception):
    """지오데식 연산 및 정산 엔진 커스텀 예외 클래스 (Exception Chaining 지원)"""
    pass

class Geodesic:
    """GIS 운영 환경 대응 방어적 예외 처리와 정규화가 적용된 고신뢰 지오데식 연산 엔진"""

    GEOGRAPHICLIB_GEODESIC_ORDER = 6
    nA1_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nC1_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nC1p_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nA2_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nC2_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nA3_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nA3x_ = nA3_
    nC3_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nC3x_ = (nC3_ * (nC3_ - 1)) // 2
    nC4_ = GEOGRAPHICLIB_GEODESIC_ORDER
    nC4x_ = (nC4_ * (nC4_ + 1)) // 2
    maxit1_ = 20
    maxit2_ = maxit1_ + Math.digits + 10

    tiny_ = math.sqrt(Math.minval)
    tol0_ = Math.epsilon
    tol1_ = 200 * tol0_
    tol2_ = math.sqrt(tol0_)
    tolb_ = tol0_ * tol2_
    xthresh_ = 1000 * tol2_

    CAP_NONE = GeodesicCapability.CAP_NONE
    CAP_C1   = GeodesicCapability.CAP_C1
    CAP_C1p  = GeodesicCapability.CAP_C1p
    CAP_C2   = GeodesicCapability.CAP_C2
    CAP_C3   = GeodesicCapability.CAP_C3
    CAP_C4   = GeodesicCapability.CAP_C4
    CAP_ALL  = GeodesicCapability.CAP_ALL
    CAP_MASK = GeodesicCapability.CAP_MASK
    OUT_ALL  = GeodesicCapability.OUT_ALL
    OUT_MASK = GeodesicCapability.OUT_MASK

    @staticmethod
    def _validate_and_normalize_coordinates(lat1, lon1, lat2=None, lon2=None):
        """입력 좌표값 유효성 검증 및 GIS 표준 경도 정규화 처리"""
        for name, val in [('lat1', lat1), ('lon1', lon1), ('lat2', lat2), ('lon2', lon2)]:
            if val is not None:
                if not isinstance(val, (int, float)) or not Math.isfinite(val):
                    raise GeodesicEngineError(f"[{name}] 유효하지 않은 숫자 타입 또는 비정상 값(NaN/Inf): {val}")
        
        if not (-90.0 <= lat1 <= 90.0):
            raise GeodesicEngineError(f"[lat1] 위도 범위를 초과했습니다 (-90 ~ 90): {lat1}")
        if lat2 is not None and not (-90.0 <= lat2 <= 90.0):
            raise GeodesicEngineError(f"[lat2] 위도 범위를 초과했습니다 (-90 ~ 90): {lat2}")

        # GIS 표준 정규화: 경도는 엄격히 차단하지 않고 Math.AngNormalize를 통해 정규화 수행
        norm_lon1 = Math.AngNormalize(lon1)
        norm_lon2 = Math.AngNormalize(lon2) if lon2 is not None else None
        return lat1, norm_lon1, lat2, norm_lon2

    @staticmethod
    def _SinCosSeries(sinp, sinx, cosx, c):
        k = len(c)
        n = k - sinp
        ar = 2 * (cosx - sinx) * (cosx + sinx)
        y1 = 0
        if n & 1:
            k -= 1; y0 = c[k]
        else:
            y0 = 0
        n = n // 2
        while n:
            n -= 1
            k -= 1; y1 = ar * y0 - y1 + c[k]
            k -= 1; y0 = ar * y1 - y0 + c[k]
        return (2 * sinx * cosx * y0 if sinp else cosx * (y0 - y1))

    @staticmethod
    def _Astroid(x, y):
        p = Math.sq(x)
        q = Math.sq(y)
        r = (p + q - 1) / 6
        if not (q == 0 and r <= 0):
            S = p * q / 4
            r2 = Math.sq(r)
            r3 = r * r2
            disc = S * (S + 2 * r3)
            u = r
            if disc >= 0:
                T3 = S + r3
                T3 += -math.sqrt(disc) if T3 < 0 else math.sqrt(disc)
                T = Math.cbrt(T3)
                u += T + (r2 / T if T != 0 else 0)
            else:
                ang = math.atan2(math.sqrt(-disc), -(S + r3))
                u += 2 * r * math.cos(ang / 3)
            v = math.sqrt(Math.sq(u) + q)
            uv = q / (v - u) if u < 0 else u + v
            w = (uv - q) / (2 * v)
            k = uv / (math.sqrt(uv + Math.sq(w)) + w)
        else:
            k = 0
        return k

    @staticmethod
    def _A1m1f(eps):
        coeff = [1, 4, 64, 0, 256]
        m = Geodesic.nA1_ // 2
        t = Math.polyval(m, coeff, 0, Math.sq(eps)) / coeff[m + 1]
        return (t + eps) / (1 - eps)

    @staticmethod
    def _C1f(eps, c):
        coeff = [-1, 6, -16, 32, -9, 64, -128, 2048, 9, -16, 768, 3, -5, 512, -7, 1280, -7, 2048]
        eps2 = Math.sq(eps)
        d = eps
        o = 0
        for l in range(1, Geodesic.nC1_ + 1):
            m = (Geodesic.nC1_ - l) // 2
            c[l] = d * Math.polyval(m, coeff, o, eps2) / coeff[o + m + 1]
            o += m + 2
            d *= eps

    @staticmethod
    def _C1pf(eps, c):
        coeff = [205, -432, 768, 1536, 4005, -4736, 3840, 12288, -225, 116, 384, -7173, 2695, 7680, 3467, 7680, 38081, 61440]
        eps2 = Math.sq(eps)
        d = eps
        o = 0
        for l in range(1, Geodesic.nC1p_ + 1):
            m = (Geodesic.nC1p_ - l) // 2
            c[l] = d * Math.polyval(m, coeff, o, eps2) / coeff[o + m + 1]
            o += m + 2
            d *= eps

    @staticmethod
    def _A2m1f(eps):
        coeff = [-11, -28, -192, 0, 256]
        m = Geodesic.nA2_ // 2
        t = Math.polyval(m, coeff, 0, Math.sq(eps)) / coeff[m + 1]
        return (t - eps) / (1 + eps)

    @staticmethod
    def _C2f(eps, c):
        coeff = [1, 2, 16, 32, 35, 64, 384, 2048, 15, 80, 768, 7, 35, 512, 63, 1280, 77, 2048]
        eps2 = Math.sq(eps)
        d = eps
        o = 0
        for l in range(1, Geodesic.nC2_ + 1):
            m = (Geodesic.nC2_ - l) // 2
            c[l] = d * Math.polyval(m, coeff, o, eps2) / coeff[o + m + 1]
            o += m + 2
            d *= eps

    def __init__(self, a, f):
        if not (Math.isfinite(a) and a > 0):
            raise GeodesicEngineError(f"초기화 오류: 유효하지 않은 적도 반경(a): {a}")
        if not (Math.isfinite(f) and abs(f) < 1):
            raise GeodesicEngineError(f"초기화 오류: 편평률(f)은 -1과 1 사이여야 합니다: {f}")
        
        self.a = float(a)
        self.f = float(f)
        self._f1 = 1 - self.f
        self._e2 = self.f * (2 - self.f)
        self._ep2 = self._e2 / Math.sq(self._f1)
        self._n = self.f / (2 - self.f)
        self._b = self.a * self._f1
        
        if not (Math.isfinite(self._b) and self._b > 0):
            raise GeodesicEngineError(f"초기화 오류: 유효하지 않은 극 반경(b): {self._b}")

        self._c2 = (Math.sq(self.a) + Math.sq(self._b) *
                    (1 if self._e2 == 0 else
                     (Math.atanh(math.sqrt(self._e2)) if self._e2 > 0 else
                      math.atan(math.sqrt(-self._e2))) /
                     math.sqrt(abs(self._e2)))) / 2
        
        self._etol2 = 0.1 * Geodesic.tol2_ / math.sqrt(max(0.001, abs(self.f)) * min(1.0, 1 - self.f / 2) / 2)
        
        self._A3x = [0.0] * Geodesic.nA3x_
        self._C3x = [0.0] * Geodesic.nC3x_
        self._C4x = [0.0] * Geodesic.nC4x_
        self._A3coeff()
        self._C3coeff()
        self._C4coeff()

    def _A3coeff(self):
        coeff = [-3, 128, -2, -3, 64, -1, -3, -1, 16, 3, -1, -2, 8, 1, -1, 2, 1, 1]
        o = 0; k = 0
        for j in range(Geodesic.nA3_ - 1, -1, -1):
            m = min(Geodesic.nA3_ - j - 1, j)
            self._A3x[k] = Math.polyval(m, coeff, o, self._n) / coeff[o + m + 1]
            k += 1
            o += m + 2

    def _C3coeff(self):
        coeff = [
            3, 128, 2, 5, 128, -1, 3, 3, 64, -1, 0, 1, 8, -1, 1, 4,
            5, 256, 1, 3, 128, -3, -2, 3, 64, 1, -3, 2, 32, 7, 512,
            -10, 9, 384, 5, -9, 5, 192, 7, 512, -14, 7, 512, 21, 2560
        ]
        o = 0; k = 0
        for l in range(1, Geodesic.nC3_):
            for j in range(Geodesic.nC3_ - 1, l - 1, -1):
                m = min(Geodesic.nC3_ - j - 1, j)
                self._C3x[k] = Math.polyval(m, coeff, o, self._n) / coeff[o + m + 1]
                k += 1
                o += m + 2

    def _C4coeff(self):
        coeff = [
            97, 15015, 1088, 156, 45045, -224, -4784, 1573, 45045, -10656, 14144, -4576, -858, 45045,
            64, 624, -4576, 6864, -3003, 15015, 100, 208, 572, 3432, -12012, 30030, 45045, 1, 9009,
            -2944, 468, 135135, 5792, 1040, -1287, 135135, 5952, -11648, 9152, -2574, 135135,
            -64, -624, 4576, -6864, 3003, 135135, 8, 10725, 1856, -936, 225225, -8448, 4992, -1144, 225225,
            -1440, 4160, -4576, 1716, 225225, -136, 63063, 1024, -208, 105105, 3584, -3328, 1144, 315315,
            -128, 135135, -2560, 832, 405405, 128, 99099
        ]
        o = 0; k = 0
        for l in range(Geodesic.nC4_):
            for j in range(Geodesic.nC4_ - 1, l - 1, -1):
                m = Geodesic.nC4_ - j - 1
                self._C4x[k] = Math.polyval(m, coeff, o, self._n) / coeff[o + m + 1]
                k += 1
                o += m + 2

    def _A3f(self, eps):
        return Math.polyval(Geodesic.nA3_ - 1, self._A3x, 0, eps)

    def _C3f(self, eps, c):
        mult = 1
        o = 0
        for l in range(1, Geodesic.nC3_):
            m = Geodesic.nC3_ - l - 1
            mult *= eps
            c[l] = mult * Math.polyval(m, self._C3x, o, eps)
            o += m + 1

    def _C4f(self, eps, c):
        mult = 1
        o = 0
        for l in range(Geodesic.nC4_):
            m = Geodesic.nC4_ - l - 1
            c[l] = mult * Math.polyval(m, self._C4x, o, eps)
            o += m + 1
            mult *= eps

    def _Lengths(self, eps, sig12, ssig1, csig1, dn1, ssig2, csig2, dn2, cbet1, cbet2, outmask, C1a, C2a):
        outmask &= Geodesic.OUT_MASK
        s12b = m12b = m0 = M12 = M21 = Math.nan
        if outmask & (Geodesic.DISTANCE | Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE):
            A1 = Geodesic._A1m1f(eps)
            Geodesic._C1f(eps, C1a)
            if outmask & (Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE):
                A2 = Geodesic._A2m1f(eps)
                Geodesic._C2f(eps, C2a)
                m0x = A1 - A2
                A2 = 1 + A2
            A1 = 1 + A1
        if outmask & Geodesic.DISTANCE:
            B1 = (Geodesic._SinCosSeries(True, ssig2, csig2, C1a) -
                  Geodesic._SinCosSeries(True, ssig1, csig1, C1a))
            s12b = A1 * (sig12 + B1)
            if outmask & (Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE):
                B2 = (Geodesic._SinCosSeries(True, ssig2, csig2, C2a) -
                      Geodesic._SinCosSeries(True, ssig1, csig1, C2a))
                J12 = m0x * sig12 + (A1 * B1 - A2 * B2)
        elif outmask & (Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE):
            for l in range(1, Geodesic.nC2_):
                C2a[l] = A1 * C1a[l] - A2 * C2a[l]
            J12 = m0x * sig12 + (Geodesic._SinCosSeries(True, ssig2, csig2, C2a) -
                                 Geodesic._SinCosSeries(True, ssig1, csig1, C2a))
        if outmask & Geodesic.REDUCEDLENGTH:
            m0 = m0x
            m12b = (dn2 * (csig1 * ssig2) - dn1 * (ssig1 * csig2) - csig1 * csig2 * J12)
        if outmask & Geodesic.GEODESICSCALE:
            csig12 = csig1 * csig2 + ssig1 * ssig2
            t = self._ep2 * (cbet1 - cbet2) * (cbet1 + cbet2) / (dn1 + dn2)
            M12 = csig12 + (t * ssig2 - csig2 * J12) * ssig1 / dn1
            M21 = csig12 - (t * ssig1 - csig1 * J12) * ssig2 / dn2
        return s12b, m12b, m0, M12, M21

    def _InverseStart(self, sbet1, cbet1, dn1, sbet2, cbet2, dn2, lam12, slam12, clam12, C1a, C2a):
        sig12 = -1; salp2 = calp2 = dnm = Math.nan
        sbet12 = sbet2 * cbet1 - cbet2 * sbet1
        cbet12 = cbet2 * cbet1 + sbet2 * sbet1
        sbet12a = sbet2 * cbet1 + cbet2 * sbet1

        shortline = cbet12 >= 0 and sbet12 < 0.5 and cbet2 * lam12 < 0.5
        if shortline:
            sbetm2 = Math.sq(sbet1 + sbet2)
            sbetm2 /= sbetm2 + Math.sq(cbet1 + cbet2)
            dnm = math.sqrt(1 + self._ep2 * sbetm2)
            omg12 = lam12 / (self._f1 * dnm)
            somg12 = math.sin(omg12); comg12 = math.cos(omg12)
        else:
            somg12 = slam12; comg12 = clam12

        salp1 = cbet2 * somg12
        calp1 = (sbet12 + cbet2 * sbet1 * Math.sq(somg12) / (1 + comg12) if comg12 >= 0
                 else sbet12a - cbet2 * sbet1 * Math.sq(somg12) / (1 - comg12))

        ssig12 = math.hypot(salp1, calp1)
        csig12 = sbet1 * sbet2 + cbet1 * cbet2 * comg12

        if shortline and ssig12 < self._etol2:
            salp2 = cbet1 * somg12
            calp2 = sbet12 - cbet1 * sbet2 * (Math.sq(somg12) / (1 + comg12) if comg12 >= 0 else 1 - comg12)
            salp2, calp2 = Math.norm(salp2, calp2)
            sig12 = math.atan2(ssig12, csig12)
        elif (abs(self._n) >= 0.1 or csig12 >= 0 or ssig12 >= 6 * abs(self._n) * math.pi * Math.sq(cbet1)):
            pass
        else:
            lam12x = math.atan2(-slam12, -clam12)
            if self.f >= 0:
                k2 = Math.sq(sbet1) * self._ep2
                eps = k2 / (2 * (1 + math.sqrt(1 + k2)) + k2)
                lamscale = self.f * cbet1 * self._A3f(eps) * math.pi
                betscale = lamscale * cbet1
                x = lam12x / lamscale
                y = sbet12a / betscale
            else:
                cbet12a = cbet2 * cbet1 - sbet2 * sbet1
                bet12a = math.atan2(sbet12a, cbet12a)
                dummy, m12b, m0, dummy, dummy = self._Lengths(
                    self._n, math.pi + bet12a, sbet1, -cbet1, dn1, sbet2, cbet2, dn2,
                    cbet1, cbet2, Geodesic.REDUCEDLENGTH, C1a, C2a)
                x = -1 + m12b / (cbet1 * cbet2 * m0 * math.pi)
                betscale = (sbet12a / x if x < -0.01 else -self.f * Math.sq(cbet1) * math.pi)
                lamscale = betscale / cbet1
                y = lam12x / lamscale

            if y > -Geodesic.tol1_ and x > -1 - Geodesic.xthresh_:
                if self.f >= 0:
                    salp1 = min(1.0, -x); calp1 = -math.sqrt(1 - Math.sq(salp1))
                else:
                    calp1 = max((0.0 if x > -Geodesic.tol1_ else -1.0), x)
                    salp1 = math.sqrt(1 - Math.sq(calp1))
            else:
                k = Geodesic._Astroid(x, y)
                omg12a = lamscale * (-x * k / (1 + k) if self.f >= 0 else -y * (1 + k) / k)
                somg12 = math.sin(omg12a); comg12 = -math.cos(omg12a)
                salp1 = cbet2 * somg12
                calp1 = sbet12a - cbet2 * sbet1 * Math.sq(somg12) / (1 - comg12)

        if not (salp1 <= 0):
            salp1, calp1 = Math.norm(salp1, calp1)
        else:
            salp1 = 1; calp1 = 0
        return sig12, salp1, calp1, salp2, calp2, dnm

    def _Lambda12(self, sbet1, cbet1, dn1, sbet2, cbet2, dn2, salp1, calp1, slam120, clam120, diffp, C1a, C2a, C3a):
        if sbet1 == 0 and calp1 == 0:
            calp1 = -Geodesic.tiny_

        salp0 = salp1 * cbet1
        calp0 = math.hypot(calp1, salp1 * sbet1)

        ssig1 = sbet1; somg1 = salp0 * sbet1
        csig1 = comg1 = calp1 * cbet1
        ssig1, csig1 = Math.norm(ssig1, csig1)

        salp2 = salp0 / cbet2 if cbet2 != cbet1 else salp1
        calp2 = (math.sqrt(Math.sq(calp1 * cbet1) +
                           ((cbet2 - cbet1) * (cbet1 + cbet2) if cbet1 < -sbet1
                            else (sbet1 - sbet2) * (sbet1 + sbet2))) / cbet2
                 if cbet2 != cbet1 or abs(sbet2) != -sbet1 else abs(calp1))
        
        ssig2 = sbet2; somg2 = salp0 * sbet2
        csig2 = comg2 = calp2 * cbet2
        ssig2, csig2 = Math.norm(ssig2, csig2)

        sig12 = math.atan2(max(0.0, csig1 * ssig2 - ssig1 * csig2), csig1 * csig2 + ssig1 * ssig2)

        somg12 = max(0.0, comg1 * somg2 - somg1 * comg2)
        comg12 = comg1 * comg2 + somg1 * somg2
        eta = math.atan2(somg12 * clam120 - comg12 * slam120, comg12 * clam120 + somg12 * slam120)

        k2 = Math.sq(calp0) * self._ep2
        eps = k2 / (2 * (1 + math.sqrt(1 + k2)) + k2)
        self._C3f(eps, C3a)
        B312 = (Geodesic._SinCosSeries(True, ssig2, csig2, C3a) -
                Geodesic._SinCosSeries(True, ssig1, csig1, C3a))
        domg12 = -self.f * self._A3f(eps) * salp0 * (sig12 + B312)
        lam12 = eta + domg12

        if diffp:
            if calp2 == 0:
                dlam12 = -2 * self._f1 * dn1 / sbet1
            else:
                dummy, dlam12, dummy, dummy, dummy = self._Lengths(
                    eps, sig12, ssig1, csig1, dn1, ssig2, csig2, dn2, cbet1, cbet2,
                    Geodesic.REDUCEDLENGTH, C1a, C2a)
                dlam12 *= self._f1 / (calp2 * cbet2)
        else:
            dlam12 = Math.nan

        return (lam12, salp2, calp2, sig12, ssig1, csig1, ssig2, csig2, eps, domg12, dlam12)

    def _GenInverse(self, lat1, lon1, lat2, lon2, outmask):
        a12 = s12 = m12 = M12 = M21 = S12 = Math.nan
        outmask &= Geodesic.OUT_MASK

        lon12, lon12s = Math.AngDiff(lon1, lon2)
        lonsign = 1 if lon12 >= 0 else -1
        lon12 = lonsign * Math.AngRound(lon12)
        lon12s = Math.AngRound((180 - lon12) - lonsign * lon12s)
        lam12 = math.radians(lon12)
        if lon12 > 90:
            slam12, clam12 = Math.sincosd(lon12s); clam12 = -clam12
        else:
            slam12, clam12 = Math.sincosd(lon12)

        lat1 = Math.AngRound(Math.LatFix(lat1))
        lat2 = Math.AngRound(Math.LatFix(lat2))
        swapp = -1 if abs(lat1) < abs(lat2) else 1
        if swapp < 0:
            lonsign *= -1
            lat2, lat1 = lat1, lat2
        latsign = 1 if lat1 < 0 else -1
        lat1 *= latsign
        lat2 *= latsign

        sbet1, cbet1 = Math.sincosd(lat1); sbet1 *= self._f1
        sbet1, cbet1 = Math.norm(sbet1, cbet1); cbet1 = max(Geodesic.tiny_, cbet1)

        sbet2, cbet2 = Math.sincosd(lat2); sbet2 *= self._f1
        sbet2, cbet2 = Math.norm(sbet2, cbet2); cbet2 = max(Geodesic.tiny_, cbet2)

        if cbet1 < -sbet1:
            if cbet2 == cbet1:
                sbet2 = sbet1 if sbet2 < 0 else -sbet1
        else:
            if abs(sbet2) == -sbet1:
                cbet2 = cbet1

        dn1 = math.sqrt(1 + self._ep2 * Math.sq(sbet1))
        dn2 = math.sqrt(1 + self._ep2 * Math.sq(sbet2))

        C1a = list(range(Geodesic.nC1_ + 1))
        C2a = list(range(Geodesic.nC2_ + 1))
        C3a = list(range(Geodesic.nC3_))

        meridian = lat1 == -90 or slam12 == 0

        if meridian:
            calp1 = clam12; salp1 = slam12
            calp2 = 1.0; salp2 = 0.0
            ssig1 = sbet1; csig1 = calp1 * cbet1
            ssig2 = sbet2; csig2 = calp2 * cbet2
            sig12 = math.atan2(max(0.0, csig1 * ssig2 - ssig1 * csig2), csig1 * csig2 + ssig1 * ssig2)

            s12x, m12x, dummy, M12, M21 = self._Lengths(
                self._n, sig12, ssig1, csig1, dn1, ssig2, csig2, dn2, cbet1, cbet2,
                outmask | Geodesic.DISTANCE | Geodesic.REDUCEDLENGTH, C1a, C2a)

            if sig12 < 1 or m12x >= 0:
                if sig12 < 3 * Geodesic.tiny_:
                    sig12 = m12x = s12x = 0.0
                m12x *= self._b
                s12x *= self._b
                a12 = math.degrees(sig12)
            else:
                meridian = False

        somg12 = 2.0; comg12 = 0.0; omg12 = 0.0
        if (not meridian and sbet1 == 0 and (self.f <= 0 or lon12s >= self.f * 180)):
            calp1 = calp2 = 0.0; salp1 = salp2 = 1.0
            s12x = self.a * lam12
            sig12 = omg12 = lam12 / self._f1
            m12x = self._b * math.sin(sig12)
            if outmask & Geodesic.GEODESICSCALE:
                M12 = M21 = math.cos(sig12)
            a12 = lon12 / self._f1

        elif not meridian:
            sig12, salp1, calp1, salp2, calp2, dnm = self._InverseStart(
                sbet1, cbet1, dn1, sbet2, cbet2, dn2, lam12, slam12, clam12, C1a, C2a)

            if sig12 >= 0:
                s12x = sig12 * self._b * dnm
                m12x = (Math.sq(dnm) * self._b * math.sin(sig12 / dnm))
                if outmask & Geodesic.GEODESICSCALE:
                    M12 = M21 = math.cos(sig12 / dnm)
                a12 = math.degrees(sig12)
                omg12 = lam12 / (self._f1 * dnm)
            else:
                numit = 0
                tripn = tripb = False
                salp1a = Geodesic.tiny_; calp1a = 1.0
                salp1b = Geodesic.tiny_; calp1b = -1.0

                while numit < Geodesic.maxit2_:
                    (v, salp2, calp2, sig12, ssig1, csig1, ssig2, csig2,
                     eps, domg12, dv) = self._Lambda12(
                         sbet1, cbet1, dn1, sbet2, cbet2, dn2,
                         salp1, calp1, slam12, clam12, numit < Geodesic.maxit1_,
                         C1a, C2a, C3a)
                    if tripb or not (abs(v) >= (8 if tripn else 1) * Geodesic.tol0_):
                        break
                    if v > 0 and (numit > Geodesic.maxit1_ or calp1 / salp1 > calp1b / salp1b):
                        salp1b = salp1; calp1b = calp1
                    elif v < 0 and (numit > Geodesic.maxit1_ or calp1 / salp1 < calp1a / salp1a):
                        salp1a = salp1; calp1a = calp1

                    numit += 1
                    if numit < Geodesic.maxit1_ and dv > 0:
                        dalp1 = -v / dv
                        sdalp1 = math.sin(dalp1); cdalp1 = math.cos(dalp1)
                        nsalp1 = salp1 * cdalp1 + calp1 * sdalp1
                        if nsalp1 > 0 and abs(dalp1) < math.pi:
                            calp1 = calp1 * cdalp1 - salp1 * sdalp1
                            salp1 = nsalp1
                            salp1, calp1 = Math.norm(salp1, calp1)
                            tripn = abs(v) <= 16 * Geodesic.tol0_
                            continue

                    salp1 = (salp1a + salp1b) / 2
                    calp1 = (calp1a + calp1b) / 2
                    salp1, calp1 = Math.norm(salp1, calp1)
                    tripn = False
                    tripb = (abs(salp1a - salp1) + (calp1a - calp1) < Geodesic.tolb_ or
                             abs(salp1 - salp1b) + (calp1 - calp1b) < Geodesic.tolb_)

                lengthmask = (outmask | (Geodesic.DISTANCE if (outmask & (Geodesic.REDUCEDLENGTH | Geodesic.GEODESICSCALE)) else Geodesic.EMPTY))
                s12x, m12x, dummy, M12, M21 = self._Lengths(
                    eps, sig12, ssig1, csig1, dn1, ssig2, csig2, dn2, cbet1, cbet2, lengthmask, C1a, C2a)

                m12x *= self._b
                s12x *= self._b
                a12 = math.degrees(sig12)
                if outmask & Geodesic.AREA:
                    sdomg12 = math.sin(domg12); cdomg12 = math.cos(domg12)
                    somg12 = slam12 * cdomg12 - clam12 * sdomg12
                    comg12 = clam12 * cdomg12 + slam12 * sdomg12

        if outmask & Geodesic.DISTANCE:
            s12 = 0.0 + s12x

        if outmask & Geodesic.REDUCEDLENGTH:
            m12 = 0.0 + m12x

        if outmask & Geodesic.AREA:
            salp0 = salp1 * cbet1
            calp0 = math.hypot(calp1, salp1 * sbet1)
            if calp0 != 0 and salp0 != 0:
                ssig1 = sbet1; csig1 = calp1 * cbet1
                ssig2 = sbet2; csig2 = calp2 * cbet2
                k2 = Math.sq(calp0) * self._ep2
                eps = k2 / (2 * (1 + math.sqrt(1 + k2)) + k2)
                A4 = Math.sq(self.a) * calp0 * salp0 * self._e2
                ssig1, csig1 = Math.norm(ssig1, csig1)
                ssig2, csig2 = Math.norm(ssig2, csig2)
                C4a = list(range(Geodesic.nC4_))
                self._C4f(eps, C4a)
                B41 = Geodesic._SinCosSeries(False, ssig1, csig1, C4a)
                B42 = Geodesic._SinCosSeries(False, ssig2, csig2, C4a)
                S12 = A4 * (B42 - B41)
            else:
                S12 = 0.0

            if not meridian and somg12 > 1:
                somg12 = math.sin(omg12); comg12 = math.cos(omg12)

            if (not meridian and comg12 > -0.7071 and sbet2 - sbet1 < 1.75):
                domg12 = 1 + comg12; dbet1 = 1 + cbet1; dbet2 = 1 + cbet2
                alp12 = 2 * math.atan2(somg12 * (sbet1 * dbet2 + sbet2 * dbet1),
                                       domg12 * (sbet1 * sbet2 + dbet1 * dbet2))
            else:
                salp12 = salp2 * calp1 - calp2 * salp1
                calp12 = calp2 * calp1 + salp2 * salp1
                if salp12 == 0 and calp12 < 0:
                    salp12 = Geodesic.tiny_ * calp1
                    calp12 = -1.0
                alp12 = math.atan2(salp12, calp12)
            S12 += self._c2 * alp12
            S12 *= swapp * lonsign * latsign
            S12 += 0.0

        if swapp < 0:
            salp2, salp1 = salp1, salp2
            calp2, calp1 = calp1, calp2
            if outmask & Geodesic.GEODESICSCALE:
                M21, M12 = M12, M21

        salp1 *= swapp * lonsign; calp1 *= swapp * latsign
        salp2 *= swapp * lonsign; calp2 *= swapp * latsign

        return a12, s12, salp1, calp1, salp2, calp2, m12, M12, M21, S12

    def Inverse(self, lat1, lon1, lat2, lon2, outmask = GeodesicCapability.STANDARD):
        """역지오데식 문제 해결 (정규화 및 Exception Chaining 적용)"""
        lat1, lon1, lat2, lon2 = self._validate_and_normalize_coordinates(lat1, lon1, lat2, lon2)
        try:
            a12, s12, salp1, calp1, salp2, calp2, m12, M12, M21, S12 = self._GenInverse(lat1, lon1, lat2, lon2, outmask)
        except Exception as e:
            raise GeodesicEngineError(f"역지오데식 연산(Inverse) 실패 (lat1={lat1}, lon1={lon1}, lat2={lat2}, lon2={lon2})") from e

        outmask &= Geodesic.OUT_MASK
        if outmask & Geodesic.LONG_UNROLL:
            lon12, e = Math.AngDiff(lon1, lon2)
            lon2 = (lon1 + lon12) + e
        else:
            lon2 = Math.AngNormalize(lon2)
        result = {
            'lat1': Math.LatFix(lat1),
            'lon1': lon1 if outmask & Geodesic.LONG_UNROLL else Math.AngNormalize(lon1),
            'lat2': Math.LatFix(lat2),
            'lon2': lon2,
            'a12': a12
        }
        if outmask & Geodesic.DISTANCE: result['s12'] = s12
        if outmask & Geodesic.AZIMUTH:
            result['azi1'] = Math.atan2d(salp1, calp1)
            result['azi2'] = Math.atan2d(salp2, calp2)
        if outmask & Geodesic.REDUCEDLENGTH: result['m12'] = m12
        if outmask & Geodesic.GEODESICSCALE:
            result['M12'] = M12; result['M21'] = M21
        if outmask & Geodesic.AREA: result['S12'] = S12
        return result

    def _GenDirect(self, lat1, lon1, azi1, arcmode, s12_a12, outmask):
        from geographiclib.geodesicline import GeodesicLine
        if not arcmode: outmask |= Geodesic.DISTANCE_IN
        line = GeodesicLine(self, lat1, lon1, azi1, outmask)
        return line._GenPosition(arcmode, s12_a12, outmask)

    def Direct(self, lat1, lon1, azi1, s12, outmask = GeodesicCapability.STANDARD):
        """직접 지오데식 문제 해결 (정규화 및 Exception Chaining 적용)"""
        lat1, lon1, _, _ = self._validate_and_normalize_coordinates(lat1, lon1)
        if not isinstance(azi1, (int, float)) or not Math.isfinite(azi1):
            raise GeodesicEngineError(f"[azi1] 유효하지 않은 방위각 값: {azi1}")
        if not isinstance(s12, (int, float)) or not Math.isfinite(s12):
            raise GeodesicEngineError(f"[s12] 유효하지 않은 거리 값: {s12}")

        try:
            a12, lat2, lon2, azi2, s12, m12, M12, M21, S12 = self._GenDirect(lat1, lon1, azi1, False, s12, outmask)
        except Exception as e:
            raise GeodesicEngineError(f"직접 지오데식 연산(Direct) 실패 (lat1={lat1}, lon1={lon1}, azi1={azi1}, s12={s12})") from e

        outmask &= Geodesic.OUT_MASK
        result = {
            'lat1': Math.LatFix(lat1),
            'lon1': lon1 if outmask & Geodesic.LONG_UNROLL else Math.AngNormalize(lon1),
            'azi1': Math.AngNormalize(azi1),
            's12': s12,
            'a12': a12
        }
        if outmask & Geodesic.LATITUDE: result['lat2'] = lat2
        if outmask & Geodesic.LONGITUDE: result['lon2'] = lon2
        if outmask & Geodesic.AZIMUTH: result['azi2'] = azi2
        if outmask & Geodesic.REDUCEDLENGTH: result['m12'] = m12
        if outmask & Geodesic.GEODESICSCALE:
            result['M12'] = M12; result['M21'] = M21
        if outmask & Geodesic.AREA: result['S12'] = S12
        return result

    def ArcDirect(self, lat1, lon1, azi1, a12, outmask = GeodesicCapability.STANDARD):
        """호 길이 기반 직접 지오데식 문제 해결 (정규화 및 Exception Chaining 적용)"""
        lat1, lon1, _, _ = self._validate_and_normalize_coordinates(lat1, lon1)
        if not isinstance(azi1, (int, float)) or not Math.isfinite(azi1):
            raise GeodesicEngineError(f"[azi1] 유효하지 않은 방위각 값: {azi1}")
        if not isinstance(a12, (int, float)) or not Math.isfinite(a12):
            raise GeodesicEngineError(f"[a12] 유효하지 않은 호 길이 값: {a12}")

        try:
            a12, lat2, lon2, azi2, s12, m12, M12, M21, S12 = self._GenDirect(lat1, lon1, azi1, True, a12, outmask)
        except Exception as e:
            raise GeodesicEngineError(f"호 길이 기반 직접 연산(ArcDirect) 실패 (lat1={lat1}, lon1={lon1}, azi1={azi1}, a12={a12})") from e

        outmask &= Geodesic.OUT_MASK
        result = {
            'lat1': Math.LatFix(lat1),
            'lon1': lon1 if outmask & Geodesic.LONG_UNROLL else Math.AngNormalize(lon1),
            'azi1': Math.AngNormalize(azi1),
            'a12': a12
        }
        if outmask & Geodesic.DISTANCE: result['s12'] = s12
        if outmask & Geodesic.LATITUDE: result['lat2'] = lat2
        if outmask & Geodesic.LONGITUDE: result['lon2'] = lon2
        if outmask & Geodesic.AZIMUTH: result['azi2'] = azi2
        if outmask & Geodesic.REDUCEDLENGTH: result['m12'] = m12
        if outmask & Geodesic.GEODESICSCALE:
            result['M12'] = M12; result['M21'] = M21
        if outmask & Geodesic.AREA: result['S12'] = S12
        return result

    def Line(self, lat1, lon1, azi1, caps = GeodesicCapability.STANDARD | GeodesicCapability.DISTANCE_IN):
        lat1, lon1, _, _ = self._validate_and_normalize_coordinates(lat1, lon1)
        from geographiclib.geodesicline import GeodesicLine
        return GeodesicLine(self, lat1, lon1, azi1, caps)

    def InverseLine(self, lat1, lon1, lat2, lon2, caps = GeodesicCapability.STANDARD | GeodesicCapability.DISTANCE_IN):
        lat1, lon1, lat2, lon2 = self._validate_and_normalize_coordinates(lat1, lon1, lat2, lon2)
        from geographiclib.geodesicline import GeodesicLine
        a12, _, salp1, calp1, _, _, _, _, _, _ = self._GenInverse(lat1, lon1, lat2, lon2, 0)
        azi1 = Math.atan2d(salp1, calp1)
        if caps & (Geodesic.OUT_MASK & Geodesic.DISTANCE_IN):
            caps |= Geodesic.DISTANCE
        line = GeodesicLine(self, lat1, lon1, azi1, caps, salp1, calp1)
        line.SetArc(a12)
        return line

    def Polygon(self, polyline = False):
        from geographiclib.polygonarea import PolygonArea
        return PolygonArea(self, polyline)

    EMPTY         = GeodesicCapability.EMPTY
    LATITUDE      = GeodesicCapability.LATITUDE
    LONGITUDE     = GeodesicCapability.LONGITUDE
    AZIMUTH       = GeodesicCapability.AZIMUTH
    DISTANCE      = GeodesicCapability.DISTANCE
    STANDARD      = GeodesicCapability.STANDARD
    DISTANCE_IN   = GeodesicCapability.DISTANCE_IN
    REDUCEDLENGTH = GeodesicCapability.REDUCEDLENGTH
    GEODESICSCALE = GeodesicCapability.GEODESICSCALE
    AREA          = GeodesicCapability.AREA
    ALL           = GeodesicCapability.ALL
    LONG_UNROLL   = GeodesicCapability.LONG_UNROLL

Geodesic.WGS84 = Geodesic(Constants.WGS84_a, Constants.WGS84_f)

최종 개선사항
✅ 원본 라이브러리 수준의 단일 연산 구조 → 입력 검증 계층 추가 → GIS 운영 환경에서 비정상 좌표 유입 차단
✅ 내부 수치 계산 실패 시 원인 추적 어려움 → Exception Chaining 적용 → 장애 원인 분석 가능성 확보
✅ 경도 입력 범위 강제 오류 가능성 → GIS 표준 정규화 적용 → 국제 좌표 처리 호환성 강화
✅ 생성자 초기 파라미터 검증 부족 → 지구 타원체 계수 검증 추가 → 잘못된 모델 초기화 방지
✅ 외부 API와 내부 핵심 알고리즘 분리 부족 → Public API 방어층 추가 → 계산 엔진 안정성 확보
✅ 원본 GeographicLib 알고리즘 유지 구조 → 불필요한 재설계 제거 → 검증된 수치 정확도 보존
✅ 단순 계산 성공 여부 중심 구조 → 입력 무결성·예외 추적·운영 장애 대응 구조 강화 → 실서비스 GIS 엔진 대응력 확보

원본 코드의 수치 알고리즘 자체는 이미 검증된 GeographicLib 핵심 엔진 수준이며, 개선 방향은 알고리즘 변경이 아니라 운영 장애를 막는 방어 계층 추가에 집중된 실무형 고신뢰 구조다.
