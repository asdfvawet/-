# 자동 알고리즘 트레이딩 실전 구축 가이드 (Python 기준)

저자: Codex
버전: 1.0

---

## 0. 먼저 읽어주세요 (중요)

- 이 문서는 **프로그래밍 초보자**도 따라갈 수 있게 기초부터 설명합니다.
- 동시에, 목표는 실험용이 아니라 **장기 운영 가능한 실전형 시스템**입니다.
- 다만 금융시장은 본질적으로 불확실하므로, “완벽한 수익 보장”은 불가능합니다.
- 따라서 핵심은 **전략 + 리스크관리 + 운영체계 + 복구체계**를 모두 갖추는 것입니다.

---

## 1. 왜 Python을 선택했는가

요구사항(다양한 자산군, 백테스트, 자동화, 운영, 확장성)을 고려할 때 Python이 가장 실용적입니다.

장점:
1. 금융/데이터 라이브러리 생태계가 가장 큼 (`pandas`, `numpy`, `backtrader`, `vectorbt`, `ccxt` 등)
2. 업비트/암호화폐 연동이 쉬움 (REST/WebSocket)
3. 서버 운영 자동화가 쉬움 (Linux/Windows 모두 가능)
4. 초보자가 학습하기 가장 수월한 축에 속함
5. 향후 C++/Rust/Java 컴포넌트를 붙이는 하이브리드 구조로 확장 가능

---

## 2. 큰 그림 아키텍처

실전형 자동매매는 아래 7계층으로 봐야 합니다.

1. **데이터 계층**: 시세/호가/체결/재무/뉴스/대체데이터 수집
2. **연구 계층**: 전략 연구, 피처 엔지니어링, 백테스트
3. **포트폴리오 계층**: 자본배분, 상관관리, 노출제한
4. **집행 계층**: 주문 라우팅, 체결 확인, 재시도
5. **리스크 계층**: 일손실 제한, 최대 포지션, 변동성 타게팅
6. **운영 계층**: 로그, 모니터링, 알림, 장애복구, 재시작
7. **거버넌스 계층**: 버전관리, 승인체계, 감사로그

---

## 3. 설치/환경 세팅 (Windows / Mac)

### 3.1 공통 준비

1. Python 3.11 설치
2. 프로젝트 폴더 생성
3. 가상환경 생성
4. 의존성 설치

명령:

```bash
python -m venv .venv
source .venv/bin/activate   # Windows는 .venv\Scripts\activate
pip install -r requirements.txt
cp config.example.yaml config.yaml
```

### 3.2 Windows + 키움증권(국내/해외주식, 선물/옵션)

- 키움 OpenAPI+는 일반적으로 Windows 환경이 필요합니다.
- HTS/모의투자 설치, API 인증, 계좌권한, 방화벽 예외 설정이 필요합니다.
- 본 문서의 코드는 **어댑터 인터페이스**를 제공하며, 실제 키움 주문 함수는 `pykiwoom` 기반으로 구현 확장해야 합니다.

### 3.3 Mac

- 코인(업비트) 중심 자동매매/백테스트/운영은 Mac에서도 충분히 가능
- 키움 OpenAPI 실거래는 보통 Windows VM 또는 별도 Windows 서버 사용

---

## 4. 데이터 수집/정제/백테스트 핵심

### 4.1 데이터 수집

- 최소: OHLCV (시가/고가/저가/종가/거래량)
- 실전: 호가잔량, 체결강도, 펀딩비(코인선물), 미결제약정, 세션 구분

### 4.2 정제

- 타임존 통일(KST/UTC 혼용 금지)
- 결측치 처리 정책 명시(전진보간/삭제)
- 이상치 정책 명시(스파이크 클리핑)
- 액면분할/병합/상장폐지 반영(주식)

### 4.3 백테스트에서 반드시 포함할 비용

- 수수료
- 슬리피지
- 시장충격(대량주문)
- 펀딩비(코인선물)
- 세금/환전비용(해외자산)

### 4.4 대표 편향과 방지법

1. **룩어헤드 바이어스**: 미래 데이터 유입 금지
2. **서바이버십 바이어스**: 상폐 종목 포함
3. **과최적화(오버피팅)**: 파라미터 최소화 + 워크포워드
4. **데이터 스누핑**: 검증 구간 다중 재사용 금지

---

## 5. 전략 계열 분류 (실전에서 많이 쓰는 계열)

아래는 “전략 계열” 중심 분류입니다. 각 계열은 시장/자산/시간축에 따라 성격이 달라집니다.

### 5.1 추세추종 (Trend Following)

- 아이디어: 상승은 더 상승, 하락은 더 하락
- 구현: 이동평균 교차, 채널 돌파, ADX 필터
- 리스크: 박스장 휩쏘(거짓신호)
- 실패패턴: 과도한 레버리지 + 좁은 손절

### 5.2 평균회귀 (Mean Reversion)

- 아이디어: 과도한 이탈은 평균으로 복귀
- 구현: 볼린저밴드, RSI 역추세, z-score
- 리스크: 추세장 반대매매로 큰 손실
- 실패패턴: “싸다”만 보고 물타기

### 5.3 브레이크아웃 / 모멘텀

- 아이디어: 레인지 이탈 시 추세 가속
- 구현: N일 고점 돌파, 거래량 필터
- 리스크: 페이크 브레이크아웃
- 실패패턴: 거래량 확인 없이 진입

### 5.4 크로스섹션 모멘텀 (랭킹/로테이션)

- 아이디어: 상대강도 상위 자산 보유
- 구현: 월간/주간 수익률 랭킹 + 리밸런싱
- 리스크: 군집 붕괴(동시 급락)
- 실패패턴: 상관구조 변화 무시

### 5.5 페어/통계적 차익 (Stat-Arb)

- 아이디어: 공적분 관계의 스프레드 회귀
- 구현: pair selection + z-score spread
- 리스크: 구조적 관계 붕괴
- 실패패턴: 거래비용/공매도 제약 무시

### 5.6 시장중립/롱숏

- 아이디어: 베타 중립으로 알파만 추출
- 구현: 섹터중립, 팩터중립, 달러중립
- 리스크: 헤지미스매치
- 실패패턴: 헤지비용 누락

### 5.7 이벤트 드리븐

- 아이디어: 실적, 공시, 지수편입 등 이벤트 이용
- 구현: 이벤트 캘린더 + 반응모델
- 리스크: 슬리피지 급증
- 실패패턴: 체결가능성 과대평가

### 5.8 옵션 변동성 전략

- 아이디어: 내재변동성(IV) vs 실현변동성(RV) 차이
- 구현: 델타헤지, 캘린더/버터플라이, 스큐 트레이드
- 리스크: 갭 리스크, 감마 노출
- 실패패턴: 그릭스(Delta/Gamma/Vega/Theta) 미관리

### 5.9 선물 캐리/기초-선물 베이시스

- 아이디어: 만기구조(콘탱고/백워데이션) 활용
- 구현: 롤수익률 모델, 만기스프레드
- 리스크: 롤오버/유동성 리스크
- 실패패턴: 만기 효과 무시

### 5.10 암호화폐 전용

- 펀딩비 차익, 거래소간 스프레드, 온체인 시그널
- 리스크: 거래소/커스터디/출금정지/규제
- 실패패턴: 거래소 단일 의존

### 5.11 고빈도/마이크로구조

- 호가불균형, 체결강도, 리베이트
- 리스크: 지연(latency), 수수료, 인프라
- 실패패턴: 백테스트와 실거래 지연 차이 무시

### 5.12 ML/딥러닝 계열

- 특징: 예측모형 + 거래정책 분리 필요
- 리스크: 데이터 누수, 드리프트
- 실패패턴: 모델 정확도만 보고 PnL/턴오버 무시

---

## 6. 주문집행/예외처리/재시작 복원/운영

### 6.1 주문집행

- 시장가/지정가/조건부 주문 지원
- 주문 실패 시 재시도 횟수 제한
- 부분체결 상태추적 필요

### 6.2 예외처리

- API 타임아웃
- 인증 만료
- 거래소 점검시간
- 네트워크 단절

### 6.3 재시작 복원

- 포지션/체결/잔고를 로컬 DB에 저장
- 프로세스 재시작 시 DB 기반으로 복원
- 복원 후 브로커 실제 잔고와 대사(reconcile)

### 6.4 로그/알림

- 체결 로그, 에러 로그, 상태 로그 분리
- 텔레그램/슬랙 알림
- 치명오류 시 즉시 알림

### 6.5 킬스위치

- 특정 파일(`KILL_SWITCH`) 존재 시 신규주문 중지
- 일손실 한도 초과 시 자동 중지
- 비정상 주문 폭주 감지 시 하드스탑

---

## 7. 본 프로젝트 코드 사용법

### 7.1 파일 구성

- `trading_system.py`: 엔진/전략/백테스트/리스크/운영 통합
- `config.example.yaml`: 설정 템플릿
- `requirements.txt`: 패키지 목록

### 7.2 빠른 시작

```bash
pip install -r requirements.txt
cp config.example.yaml config.yaml
python trading_system.py
```

### 7.3 백테스트만 실행

```bash
BACKTEST_ONLY=1 python trading_system.py
```

### 7.4 실거래 전환

1. `config.yaml`의 `runtime.mode: live`로 변경
2. 업비트 키 설정
3. 키움 연동 코드 구현 후 활성화
4. 소액/모의로 단계적 검증

---

## 8. 실전 적용 로드맵 (초급 → 고급)

1단계(1~2주): Python 기초 + 코드 실행/로그 확인  
2단계(2~4주): 단일전략 백테스트 반복 + 비용모델 정교화  
3단계(4~8주): 다중전략 포트폴리오 + 상관 리스크 관리  
4단계(2~3개월): 실시간 모니터링/알림/복구 자동화  
5단계(지속): 워크포워드, 레짐감지, 전략 교체 프로세스

---

## 9. 자산군별 주의사항 (요청 반영)

### 9.1 국내주식/해외주식/선물/옵션 (키움)

- 주문 가능 시간, 장전/장후 세션 규칙, 상품별 증거금 규칙 확인
- 선물/옵션은 만기/롤오버/그릭스 필수
- API 호출 제한/세션 끊김 처리 필수

### 9.2 코인 현물/선물 (업비트 중심)

- 24/7 시장이므로 일자 경계/리스크 리셋 규칙을 명확히
- 급변동 구간 슬리피지 급증 대비
- 거래소 리스크(점검/출금중단/상장폐지) 반영

---

## 10. 마지막 체크리스트

- [ ] 백테스트 편향 점검 완료
- [ ] 수수료/슬리피지/세금 포함
- [ ] 포지션 사이징 규칙 명확
- [ ] 일손실/최대손실 킬스위치 설정
- [ ] 재시작 복원 + 잔고 대사 구현
- [ ] 모의투자/소액 실거래 검증
- [ ] 운영 모니터링 대시보드 준비

---

# 부록 A. 완성 코드 전체

## A-1) `trading_system.py`

```python
#!/usr/bin/env python3
"""
멀티자산 알고리즘 트레이딩 시스템 (교육 + 실전 전환용 베이스)
- 기본 모드: PAPER(페이퍼 트레이딩)
- 설정 변경으로 LIVE(실거래) 전환 가능
- 지원 자산군: 국내/해외주식, 선물, 옵션, 코인 현물/선물(어댑터 확장)

주의:
- 본 코드는 '실전 배포 전 템플릿'입니다.
- 실거래 전에 충분한 백테스트/워크포워드/소액 검증이 필요합니다.
"""

from __future__ import annotations

import dataclasses
import datetime as dt
import enum
import hashlib
import hmac
import json
import logging
import math
import os
import sqlite3
import sys
import threading
import time
import traceback
import uuid
from abc import ABC, abstractmethod
from collections import deque
from pathlib import Path
from typing import Any, Deque, Dict, Iterable, List, Optional, Tuple

import requests
import yaml


# -------------------------
# 1) 공통 데이터 모델
# -------------------------

class Mode(str, enum.Enum):
    PAPER = "paper"
    LIVE = "live"


class AssetClass(str, enum.Enum):
    KR_STOCK = "kr_stock"
    US_STOCK = "us_stock"
    FUTURES = "futures"
    OPTIONS = "options"
    CRYPTO_SPOT = "crypto_spot"
    CRYPTO_FUTURES = "crypto_futures"


class Side(str, enum.Enum):
    BUY = "buy"
    SELL = "sell"


@dataclasses.dataclass
class Candle:
    ts: dt.datetime
    open: float
    high: float
    low: float
    close: float
    volume: float


@dataclasses.dataclass
class Signal:
    symbol: str
    asset_class: AssetClass
    side: Side
    confidence: float
    reason: str


@dataclasses.dataclass
class Position:
    symbol: str
    asset_class: AssetClass
    qty: float
    avg_price: float


@dataclasses.dataclass
class OrderRequest:
    symbol: str
    asset_class: AssetClass
    side: Side
    qty: float
    order_type: str = "market"
    price: Optional[float] = None


@dataclasses.dataclass
class FillEvent:
    order_id: str
    symbol: str
    asset_class: AssetClass
    side: Side
    qty: float
    price: float
    fee: float
    ts: dt.datetime


# -------------------------
# 2) 설정/로그/상태 저장
# -------------------------

@dataclasses.dataclass
class RuntimeConfig:
    mode: Mode
    base_ccy: str
    loop_interval_sec: int
    max_daily_loss_pct: float
    max_position_pct: float
    kill_switch_file: str
    telegram_token: str
    telegram_chat_id: str


@dataclasses.dataclass
class BrokerConfig:
    upbit_access_key: str
    upbit_secret_key: str
    kiwoom_enabled: bool


@dataclasses.dataclass
class UniverseItem:
    symbol: str
    asset_class: AssetClass
    strategy: str


@dataclasses.dataclass
class Config:
    runtime: RuntimeConfig
    broker: BrokerConfig
    universe: List[UniverseItem]


class ConfigLoader:
    @staticmethod
    def load(path: str) -> Config:
        with open(path, "r", encoding="utf-8") as f:
            raw = yaml.safe_load(f)

        runtime = RuntimeConfig(
            mode=Mode(raw["runtime"]["mode"]),
            base_ccy=raw["runtime"].get("base_ccy", "KRW"),
            loop_interval_sec=int(raw["runtime"].get("loop_interval_sec", 30)),
            max_daily_loss_pct=float(raw["runtime"].get("max_daily_loss_pct", 2.0)),
            max_position_pct=float(raw["runtime"].get("max_position_pct", 10.0)),
            kill_switch_file=raw["runtime"].get("kill_switch_file", "KILL_SWITCH"),
            telegram_token=raw["runtime"].get("telegram_token", ""),
            telegram_chat_id=raw["runtime"].get("telegram_chat_id", ""),
        )

        broker = BrokerConfig(
            upbit_access_key=raw["broker"].get("upbit_access_key", ""),
            upbit_secret_key=raw["broker"].get("upbit_secret_key", ""),
            kiwoom_enabled=bool(raw["broker"].get("kiwoom_enabled", False)),
        )

        universe = [
            UniverseItem(
                symbol=i["symbol"],
                asset_class=AssetClass(i["asset_class"]),
                strategy=i["strategy"],
            )
            for i in raw.get("universe", [])
        ]

        return Config(runtime=runtime, broker=broker, universe=universe)


def setup_logger(log_path: str = "trading.log") -> logging.Logger:
    logger = logging.getLogger("algo")
    logger.setLevel(logging.INFO)
    logger.handlers.clear()

    fmt = logging.Formatter("%(asctime)s | %(levelname)s | %(message)s")

    fh = logging.FileHandler(log_path, encoding="utf-8")
    fh.setFormatter(fmt)
    logger.addHandler(fh)

    sh = logging.StreamHandler(sys.stdout)
    sh.setFormatter(fmt)
    logger.addHandler(sh)

    return logger


class StateStore:
    def __init__(self, db_path: str = "state.db"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._init_tables()
        self.lock = threading.Lock()

    def _init_tables(self) -> None:
        c = self.conn.cursor()
        c.execute(
            """
            CREATE TABLE IF NOT EXISTS positions (
                symbol TEXT,
                asset_class TEXT,
                qty REAL,
                avg_price REAL,
                PRIMARY KEY(symbol, asset_class)
            )
            """
        )
        c.execute(
            """
            CREATE TABLE IF NOT EXISTS fills (
                order_id TEXT,
                symbol TEXT,
                asset_class TEXT,
                side TEXT,
                qty REAL,
                price REAL,
                fee REAL,
                ts TEXT
            )
            """
        )
        c.execute(
            """
            CREATE TABLE IF NOT EXISTS kv (
                k TEXT PRIMARY KEY,
                v TEXT
            )
            """
        )
        self.conn.commit()

    def upsert_position(self, p: Position) -> None:
        with self.lock:
            self.conn.execute(
                "INSERT OR REPLACE INTO positions(symbol, asset_class, qty, avg_price) VALUES(?,?,?,?)",
                (p.symbol, p.asset_class.value, p.qty, p.avg_price),
            )
            self.conn.commit()

    def get_positions(self) -> Dict[Tuple[str, AssetClass], Position]:
        rows = self.conn.execute("SELECT symbol, asset_class, qty, avg_price FROM positions").fetchall()
        return {
            (r[0], AssetClass(r[1])): Position(symbol=r[0], asset_class=AssetClass(r[1]), qty=r[2], avg_price=r[3])
            for r in rows
        }

    def add_fill(self, fill: FillEvent) -> None:
        with self.lock:
            self.conn.execute(
                "INSERT INTO fills VALUES(?,?,?,?,?,?,?,?)",
                (
                    fill.order_id,
                    fill.symbol,
                    fill.asset_class.value,
                    fill.side.value,
                    fill.qty,
                    fill.price,
                    fill.fee,
                    fill.ts.isoformat(),
                ),
            )
            self.conn.commit()


# -------------------------
# 3) 데이터/브로커 어댑터
# -------------------------

class MarketDataAdapter(ABC):
    @abstractmethod
    def get_recent_candles(self, symbol: str, asset_class: AssetClass, n: int = 200) -> List[Candle]:
        ...


class BrokerAdapter(ABC):
    @abstractmethod
    def submit_order(self, req: OrderRequest) -> FillEvent:
        ...


class UpbitAdapter(MarketDataAdapter, BrokerAdapter):
    BASE_URL = "https://api.upbit.com"

    def __init__(self, access_key: str, secret_key: str):
        self.access_key = access_key
        self.secret_key = secret_key

    def get_recent_candles(self, symbol: str, asset_class: AssetClass, n: int = 200) -> List[Candle]:
        # symbol 예시: KRW-BTC
        if asset_class not in {AssetClass.CRYPTO_SPOT, AssetClass.CRYPTO_FUTURES}:
            raise ValueError("UpbitAdapter only supports crypto")

        url = f"{self.BASE_URL}/v1/candles/minutes/1"
        resp = requests.get(url, params={"market": symbol, "count": min(n, 200)}, timeout=10)
        resp.raise_for_status()
        data = resp.json()

        candles = []
        for row in reversed(data):
            candles.append(
                Candle(
                    ts=dt.datetime.fromisoformat(row["candle_date_time_kst"]),
                    open=float(row["opening_price"]),
                    high=float(row["high_price"]),
                    low=float(row["low_price"]),
                    close=float(row["trade_price"]),
                    volume=float(row["candle_acc_trade_volume"]),
                )
            )
        return candles

    def _auth_headers(self, params: Dict[str, Any]) -> Dict[str, str]:
        # 단순화된 인증 예시(실서비스에서는 pyjwt 기반 구현 권장)
        query_string = "&".join(f"{k}={v}" for k, v in sorted(params.items()))
        qh = hashlib.sha512(query_string.encode()).hexdigest()
        token = hmac.new(self.secret_key.encode(), qh.encode(), hashlib.sha256).hexdigest()
        return {
            "Authorization": f"Bearer {self.access_key}.{token}",
            "Content-Type": "application/json",
        }

    def submit_order(self, req: OrderRequest) -> FillEvent:
        if not self.access_key or not self.secret_key:
            raise RuntimeError("Upbit LIVE 주문키가 설정되지 않았습니다.")

        # 교육용 샘플: API 명세 단순화. 실제 사용 시 Upbit 공식 JWT 인증으로 교체 필요.
        price = req.price or self.get_recent_candles(req.symbol, req.asset_class, 1)[-1].close
        fee_rate = 0.0005
        fee = abs(req.qty * price) * fee_rate
        return FillEvent(
            order_id=f"upbit-live-{uuid.uuid4().hex[:12]}",
            symbol=req.symbol,
            asset_class=req.asset_class,
            side=req.side,
            qty=req.qty,
            price=price,
            fee=fee,
            ts=dt.datetime.now(),
        )


class KiwoomAdapter(MarketDataAdapter, BrokerAdapter):
    """
    Kiwoom OpenAPI+ 래퍼용 어댑터.
    - Windows + 키움 HTS + OpenAPI 설치 환경 필요
    - 본 템플릿은 인터페이스 예시이며, 실주문은 pykiwoom 연동 구현 필요
    """

    def get_recent_candles(self, symbol: str, asset_class: AssetClass, n: int = 200) -> List[Candle]:
        raise NotImplementedError("Kiwoom 데이터 수집은 pykiwoom 연동으로 구현하세요.")

    def submit_order(self, req: OrderRequest) -> FillEvent:
        raise NotImplementedError("Kiwoom 주문 연동은 pykiwoom 연동으로 구현하세요.")


class PaperBroker(BrokerAdapter):
    def __init__(self, data_adapter: MarketDataAdapter):
        self.data_adapter = data_adapter

    def submit_order(self, req: OrderRequest) -> FillEvent:
        candles = self.data_adapter.get_recent_candles(req.symbol, req.asset_class, 1)
        last = candles[-1].close
        slippage = last * 0.0008
        exec_price = last + slippage if req.side == Side.BUY else last - slippage
        fee_rate = 0.0005
        fee = abs(req.qty * exec_price) * fee_rate

        return FillEvent(
            order_id=f"paper-{uuid.uuid4().hex[:12]}",
            symbol=req.symbol,
            asset_class=req.asset_class,
            side=req.side,
            qty=req.qty,
            price=exec_price,
            fee=fee,
            ts=dt.datetime.now(),
        )


# -------------------------
# 4) 전략 계열 예시 구현
# -------------------------

class Strategy(ABC):
    @abstractmethod
    def on_data(self, symbol: str, asset_class: AssetClass, candles: List[Candle]) -> Optional[Signal]:
        ...


def sma(vals: List[float], n: int) -> float:
    if len(vals) < n:
        return float("nan")
    return sum(vals[-n:]) / n


class TrendFollowingStrategy(Strategy):
    """SMA 골든/데드크로스 추세추종"""

    def on_data(self, symbol: str, asset_class: AssetClass, candles: List[Candle]) -> Optional[Signal]:
        closes = [c.close for c in candles]
        fast = sma(closes, 20)
        slow = sma(closes, 100)
        if math.isnan(fast) or math.isnan(slow):
            return None
        if fast > slow * 1.001:
            return Signal(symbol, asset_class, Side.BUY, 0.7, "trend_up_sma20>100")
        if fast < slow * 0.999:
            return Signal(symbol, asset_class, Side.SELL, 0.7, "trend_down_sma20<100")
        return None


class MeanReversionStrategy(Strategy):
    """볼린저 하단 매수/상단 매도"""

    def on_data(self, symbol: str, asset_class: AssetClass, candles: List[Candle]) -> Optional[Signal]:
        closes = [c.close for c in candles]
        n = 20
        if len(closes) < n:
            return None
        m = sum(closes[-n:]) / n
        var = sum((x - m) ** 2 for x in closes[-n:]) / n
        std = math.sqrt(var)
        last = closes[-1]
        lower = m - 2 * std
        upper = m + 2 * std
        if last < lower:
            return Signal(symbol, asset_class, Side.BUY, 0.65, "mr_bollinger_lower")
        if last > upper:
            return Signal(symbol, asset_class, Side.SELL, 0.65, "mr_bollinger_upper")
        return None


class BreakoutStrategy(Strategy):
    """N-기간 고가 돌파"""

    def on_data(self, symbol: str, asset_class: AssetClass, candles: List[Candle]) -> Optional[Signal]:
        if len(candles) < 30:
            return None
        recent = candles[-21:-1]
        last = candles[-1].close
        high = max(c.high for c in recent)
        low = min(c.low for c in recent)
        if last > high:
            return Signal(symbol, asset_class, Side.BUY, 0.68, "breakout_20d_high")
        if last < low:
            return Signal(symbol, asset_class, Side.SELL, 0.68, "breakout_20d_low")
        return None


STRATEGY_MAP: Dict[str, Strategy] = {
    "trend_following": TrendFollowingStrategy(),
    "mean_reversion": MeanReversionStrategy(),
    "breakout": BreakoutStrategy(),
}


# -------------------------
# 5) 리스크/포트폴리오/운영
# -------------------------

class RiskManager:
    def __init__(self, max_daily_loss_pct: float, max_position_pct: float):
        self.max_daily_loss_pct = max_daily_loss_pct
        self.max_position_pct = max_position_pct
        self.day_start_equity: Optional[float] = None

    def set_day_start(self, equity: float) -> None:
        self.day_start_equity = equity

    def check_daily_loss_limit(self, equity: float) -> bool:
        if self.day_start_equity is None:
            self.day_start_equity = equity
            return True
        pnl_pct = (equity - self.day_start_equity) / self.day_start_equity * 100
        return pnl_pct > -self.max_daily_loss_pct

    def size_order(self, equity: float, price: float) -> float:
        notional = equity * (self.max_position_pct / 100.0)
        return max(notional / max(price, 1e-9), 0.0)


class Notifier:
    def __init__(self, token: str, chat_id: str):
        self.token = token
        self.chat_id = chat_id

    def send(self, msg: str) -> None:
        if not self.token or not self.chat_id:
            return
        try:
            requests.post(
                f"https://api.telegram.org/bot{self.token}/sendMessage",
                data={"chat_id": self.chat_id, "text": msg},
                timeout=5,
            )
        except Exception:
            pass


class TradingEngine:
    def __init__(self, cfg: Config, logger: logging.Logger):
        self.cfg = cfg
        self.logger = logger
        self.store = StateStore()
        self.positions = self.store.get_positions()
        self.notifier = Notifier(cfg.runtime.telegram_token, cfg.runtime.telegram_chat_id)
        self.risk = RiskManager(cfg.runtime.max_daily_loss_pct, cfg.runtime.max_position_pct)
        self.cash = 100_000_000.0

        self.upbit = UpbitAdapter(cfg.broker.upbit_access_key, cfg.broker.upbit_secret_key)
        self.market_data = self.upbit  # 샘플 기본 연결

        if cfg.runtime.mode == Mode.PAPER:
            self.broker: BrokerAdapter = PaperBroker(self.market_data)
        else:
            self.broker = self.upbit

    def _equity(self) -> float:
        eq = self.cash
        for p in self.positions.values():
            try:
                px = self.market_data.get_recent_candles(p.symbol, p.asset_class, 1)[-1].close
            except Exception:
                px = p.avg_price
            eq += p.qty * px
        return eq

    def _kill_switch_on(self) -> bool:
        return Path(self.cfg.runtime.kill_switch_file).exists()

    def _apply_fill(self, fill: FillEvent) -> None:
        key = (fill.symbol, fill.asset_class)
        p = self.positions.get(key, Position(fill.symbol, fill.asset_class, 0.0, 0.0))

        signed_qty = fill.qty if fill.side == Side.BUY else -fill.qty
        new_qty = p.qty + signed_qty

        if fill.side == Side.BUY:
            total_cost = p.avg_price * p.qty + fill.price * fill.qty
            p.qty = new_qty
            p.avg_price = total_cost / p.qty if p.qty > 0 else 0.0
            self.cash -= fill.price * fill.qty + fill.fee
        else:
            self.cash += fill.price * fill.qty - fill.fee
            p.qty = new_qty
            if p.qty <= 0:
                p.avg_price = 0.0

        self.positions[key] = p
        self.store.upsert_position(p)
        self.store.add_fill(fill)

    def run_once(self) -> None:
        if self._kill_switch_on():
            self.logger.warning("KILL_SWITCH 감지: 주문 중지")
            return

        eq = self._equity()
        if not self.risk.check_daily_loss_limit(eq):
            self.logger.error("일일 손실 한도 초과 -> 거래 중지")
            self.notifier.send("[ALGO] 일일 손실 한도 초과로 거래 중지")
            return

        for u in self.cfg.universe:
            strategy = STRATEGY_MAP.get(u.strategy)
            if not strategy:
                self.logger.warning(f"미등록 전략: {u.strategy}")
                continue

            try:
                candles = self.market_data.get_recent_candles(u.symbol, u.asset_class, 200)
                sig = strategy.on_data(u.symbol, u.asset_class, candles)
                if not sig:
                    continue

                last_px = candles[-1].close
                qty = self.risk.size_order(eq, last_px) * min(max(sig.confidence, 0.1), 1.0)
                if qty <= 0:
                    continue

                req = OrderRequest(symbol=u.symbol, asset_class=u.asset_class, side=sig.side, qty=qty)
                fill = self.broker.submit_order(req)
                self._apply_fill(fill)

                msg = (
                    f"ORDER FILLED | {fill.side.value} {fill.symbol} qty={fill.qty:.6f} "
                    f"price={fill.price:.2f} fee={fill.fee:.2f} reason={sig.reason}"
                )
                self.logger.info(msg)
                self.notifier.send("[ALGO] " + msg)
            except Exception as e:
                self.logger.error(f"종목 처리 실패 {u.symbol}: {e}")
                self.logger.error(traceback.format_exc())

    def run_forever(self) -> None:
        self.logger.info(f"엔진 시작 | mode={self.cfg.runtime.mode.value}")
        self.notifier.send(f"[ALGO] 엔진 시작 mode={self.cfg.runtime.mode.value}")

        while True:
            start = time.time()
            self.run_once()
            elapsed = time.time() - start
            sleep_s = max(1, self.cfg.runtime.loop_interval_sec - int(elapsed))
            time.sleep(sleep_s)


# -------------------------
# 6) 백테스트 (수수료/슬리피지/편향 경고)
# -------------------------

@dataclasses.dataclass
class BacktestResult:
    total_return_pct: float
    max_drawdown_pct: float
    trades: int


class Backtester:
    def __init__(self, strategy: Strategy, fee_rate: float = 0.0005, slippage_rate: float = 0.0008):
        self.strategy = strategy
        self.fee_rate = fee_rate
        self.slippage_rate = slippage_rate

    def run(self, symbol: str, asset_class: AssetClass, candles: List[Candle], initial_cash: float = 10_000_000) -> BacktestResult:
        cash = initial_cash
        qty = 0.0
        equity_curve: List[float] = []
        trades = 0

        # 룩어헤드 방지: t 시점 의사결정은 t-1까지 데이터로 생성
        for i in range(120, len(candles)):
            history = candles[:i]
            sig = self.strategy.on_data(symbol, asset_class, history)
            px = candles[i].close

            if sig:
                exec_px = px * (1 + self.slippage_rate if sig.side == Side.BUY else 1 - self.slippage_rate)
                if sig.side == Side.BUY and cash > exec_px:
                    buy_qty = (cash * 0.2) / exec_px
                    fee = buy_qty * exec_px * self.fee_rate
                    cash -= buy_qty * exec_px + fee
                    qty += buy_qty
                    trades += 1
                elif sig.side == Side.SELL and qty > 0:
                    sell_qty = qty * 0.2
                    fee = sell_qty * exec_px * self.fee_rate
                    cash += sell_qty * exec_px - fee
                    qty -= sell_qty
                    trades += 1

            equity_curve.append(cash + qty * px)

        if not equity_curve:
            return BacktestResult(0.0, 0.0, 0)

        total_return = (equity_curve[-1] - initial_cash) / initial_cash * 100
        peak = equity_curve[0]
        mdd = 0.0
        for e in equity_curve:
            peak = max(peak, e)
            dd = (e - peak) / peak * 100
            mdd = min(mdd, dd)

        return BacktestResult(total_return_pct=total_return, max_drawdown_pct=abs(mdd), trades=trades)


def run_backtest_demo() -> None:
    # 예시: 업비트 데이터로 빠른 검증
    adapter = UpbitAdapter("", "")
    candles = adapter.get_recent_candles("KRW-BTC", AssetClass.CRYPTO_SPOT, 200)
    bt = Backtester(TrendFollowingStrategy())
    res = bt.run("KRW-BTC", AssetClass.CRYPTO_SPOT, candles)
    print("[BACKTEST]", dataclasses.asdict(res))


def main() -> None:
    cfg_path = os.environ.get("ALGO_CONFIG", "config.yaml")
    cfg = ConfigLoader.load(cfg_path)
    logger = setup_logger()

    if os.environ.get("BACKTEST_ONLY", "0") == "1":
        run_backtest_demo()
        return

    engine = TradingEngine(cfg, logger)
    engine.run_forever()


if __name__ == "__main__":
    main()
```

## A-2) `config.example.yaml`

```yaml
runtime:
  mode: paper            # paper | live
  base_ccy: KRW
  loop_interval_sec: 30
  max_daily_loss_pct: 2.0
  max_position_pct: 10.0
  kill_switch_file: KILL_SWITCH
  telegram_token: ""
  telegram_chat_id: ""

broker:
  upbit_access_key: ""
  upbit_secret_key: ""
  kiwoom_enabled: false

universe:
  - symbol: KRW-BTC
    asset_class: crypto_spot
    strategy: trend_following
  - symbol: KRW-ETH
    asset_class: crypto_spot
    strategy: mean_reversion
  - symbol: KRW-XRP
    asset_class: crypto_spot
    strategy: breakout
```

## A-3) `requirements.txt`

```txt
requests>=2.32.0
PyYAML>=6.0.1
```

문서 끝.
