# QuantConnect LEAN 엔진 심층 조사 보고서

코드베이스 전체 분석 + 웹 최신 자료까지 종합한 전범위 설명입니다.

---

## 1. LEAN이란?

LEAN은 QuantConnect가 개발한 **오픈소스 이벤트 드리븐 알고리즘 트레이딩 엔진**입니다. C# (.NET) 기반이지만 **PythonNet**을 통해 Python API를 완전히 노출합니다. 백테스팅, 리서치, 페이퍼/라이브 트레이딩을 하나의 파이프라인으로 처리합니다.

- **라이선스**: Apache 2.0 (오픈소스)
- **언어**: C# (엔진 코어) + Python (알고리즘 작성)
- **자산군**: Equity, Option, Future, FutureOption, Forex, Crypto, CryptoFuture, Index, CFD (9종)
- **브로커리지**: 30+ (IBKR, Alpaca, Schwab, Tradier, OANDA, Binance, Coinbase, Bybit, Kraken, dYdX 등)
- **철학**: 모든 컴포넌트는 **인터페이스 기반 플러그인**으로 교체 가능

> 2026년 기준 QuantConnect 클라우드는 "Agentic Studio" (LLM 기반 자동 알파 발굴)와 이더리움/솔라나 노드 네이티브 통합을 추가해 온체인 차익거래까지 커버합니다.

---

## 2. 실행 흐름 (Launcher → Engine → AlgorithmManager → 사용자 알고리즘)

```
[Launcher/Program.cs]
   ↓ config.json 로드, 핸들러 선택
[Engine/Engine.cs]
   ↓ Setup → Brokerage → DataManager → Synchronizer 구성
[Engine/AlgorithmManager.cs]
   ↓ TimeSlice 스트리밍 루프 (이벤트 디스패치)
[Algorithm/QCAlgorithm.cs (or 사용자 .py)]
   ↓ OnData / OnOrderEvent / 스케줄 콜백
[Engine/TransactionHandlers] → [Brokerages/*] → 시장
```

핵심 파일:
- [Launcher/Program.cs](Launcher/Program.cs) — 실행 진입점
- [Engine/Engine.cs](Engine/Engine.cs) — 엔진 오케스트레이터
- [Engine/AlgorithmManager.cs](Engine/AlgorithmManager.cs) — 메인 이벤트 루프
- [Engine/LeanEngineSystemHandlers.cs](Engine/LeanEngineSystemHandlers.cs), [Engine/LeanEngineAlgorithmHandlers.cs](Engine/LeanEngineAlgorithmHandlers.cs) — 핸들러 컴포지션

---

## 3. 핸들러 패턴 (LEAN의 심장)

LEAN의 모든 확장성은 **인터페이스 기반 합성(composition)** 에서 나옵니다. `config.json`의 `environment` 설정으로 핸들러 세트가 결정됩니다.

| 인터페이스 | 백테스팅 구현 | 라이브 구현 |
|---|---|---|
| `IDataFeed` | `FileSystemDataFeed` | `LiveTradingDataFeed` |
| `ITransactionHandler` | `BacktestingTransactionHandler` | `BrokerageTransactionHandler` |
| `IResultHandler` | `BacktestingResultHandler` | `LiveTradingResultHandler` |
| `IRealTimeHandler` | `BacktestingRealTimeHandler` | `LiveTradingRealTimeHandler` |
| `ISetupHandler` | `BacktestingSetupHandler` | `BrokerageSetupHandler` |
| `IHistoryProvider` | `SubscriptionDataReaderHistoryProvider` | `BrokerageHistoryProvider` |

시스템 레벨 핸들러: `IApi`, `IMessagingHandler`, `IJobQueueHandler`, `ILeanManager`.
알고리즘 레벨 부가 핸들러: `IMapFileProvider`, `IFactorFileProvider`, `IDataProvider`, `IDataCacheProvider`, `IObjectStore`, `IDataPermissionManager`.

---

## 4. 알고리즘 계층

### 4.1 QCAlgorithm (베이스 클래스)
[Algorithm/QCAlgorithm.cs](Algorithm/QCAlgorithm.cs) — partial 클래스로 분할되어 있음:
- `QCAlgorithm.Trading.cs` — 주문 API (`Order`, `SetHoldings`, `ComboMarketOrder`)
- `QCAlgorithm.Universe.cs` — 유니버스 선택
- `QCAlgorithm.History.cs` — 히스토리 요청
- `QCAlgorithm.Indicators.cs` — 지표 헬퍼 (`SMA`, `RSI`, `BB` 등)
- `QCAlgorithm.Python.cs` — Python 오버로드
- `QCAlgorithm.Framework.cs` — 프레임워크 모델 API

### 4.2 Algorithm Framework (모듈형 전략 구조)
표준화된 **다섯 모듈 파이프라인**:

```
Universe Selection → Alpha → Portfolio Construction → Risk Management → Execution
       ↓              ↓              ↓                        ↓                  ↓
    Symbol[]      Insight[]     PortfolioTarget[]      PortfolioTarget[]    Orders
```

위치: [Algorithm.Framework/](Algorithm.Framework/)
- **Alphas/** — `IAlphaModel`, `Update(Slice) → Insight[]` 반환. 예: `EmaCrossAlphaModel`, `HistoricalReturnsAlphaModel`
- **Portfolio/** — `EqualWeightingPortfolioConstructionModel`, `BlackLittermanOptimizationPortfolioConstructionModel`, `RiskParityPortfolioConstructionModel`
- **Risk/** — `MaximumDrawdownPercentModel`, `MaximumLossModel`
- **Execution/** — `ImmediateExecutionModel`, `VolumeWeightedAveragePriceExecutionModel`, `TWAPExecutionModel`
- **Selection/** — 유니버스 셀렉터 (coarse/fine, option chain, future chain)

### 4.3 Python 지원
[Algorithm.Python/](Algorithm.Python/) — 438개의 예제 `.py` 알고리즘. **Python.NET v2.0.53** 으로 CLR ↔ Python 브릿징. [ci_build_stubs.sh](ci_build_stubs.sh) 가 IDE 자동완성용 `.pyi` 스텁을 생성합니다.

---

## 5. 데이터 계층

### 5.1 BaseData 계층
[Common/Data/](Common/) — `IBaseData` 구현체: `TradeBar`, `QuoteBar`, `Tick`, `OptionChain`, `FutureChain`, `Fundamental`, 사용자 정의 `BaseData`.

### 5.2 구독 시스템
[Engine/DataFeeds/](Engine/):
- `Subscription` — 단일 심볼 데이터 스트림
- `DataManager` — 전체 구독 수명주기 관리 (commit 7602c5bde에서 유효하지 않은 구독 타입 생성 버그 수정됨)
- `Synchronizer` — 시간 정렬 (백테스트: 결정론적 / 라이브: 비동기)
- Enumerators 체인: `FillForwardEnumerator`, `SortEnumerator`, `MappingEventProvider`, `SplitEventProvider`, `DividendEventProvider`, `PriceScaleFactorEnumerator`

### 5.3 데이터 파일 포맷 (LEAN 규약)
```
data/
├── equity/usa/{tick,second,minute,hour,daily}/aapl.zip
├── option/usa/minute/AAPL.zip
├── future/usa/minute/ES.zip
├── forex/oanda/minute/EURUSD.zip
├── crypto/minute/BTCUSD.zip
├── map_files/ (심볼 매핑: 티커 변경, 상폐)
└── factor_files/ (배당·분할 조정 팩터)
```
`.zip` 안에 일자별 `.csv` — OHLCV 또는 Bid/Ask 4-OHLC.

### 5.4 HistoryProvider
`History(symbol, timespan, resolution)` 호출 시:
- 백테스트 → 디스크 CSV 직접 읽기
- 라이브 → 브로커리지 API 호출 (폴백: 디스크)

---

## 6. 시큐리티 & 포트폴리오

[Common/Securities/](Common/)

- **Security** (~1300 lines) — 각 심볼의 `SecurityHolding`, `SecurityCache`, `SecurityExchange`, 모델들
- **SecurityPortfolioManager** (~1100 lines) — `TotalPortfolioValue`, `Cash`, `Margin`, `BuyingPower`, `UnrealisedProfit`
- **CashBook** — 멀티 통화 보유 및 환율 변환
- **모델 슬롯**: `FillModel`, `SlippageModel`, `FeeModel`, `BuyingPowerModel`, `SettlementModel`, `MarginCallModel`, `MarginInterestRateModel` — 모두 브로커리지별로 오버라이드 가능

### 자산군별 특화
- **Equity**: Reg-T 마진, PDT 규칙, 분할/배당 자동 조정
- **Option**: Greeks (Delta/Gamma/Vega/Theta/Rho), 미국/유럽 옵션, 멀티플라이어=100. commit e68ee853d에서 indicator 기반 옵션 프라이스 모델 추가
- **Future**: 연속 계약(continuous contract), SPAN 마진, 롤오버
- **FutureOption**: /ES 옵션 등
- **Forex**: 핍 계산, 24/5
- **Crypto / CryptoFuture**: 분수 수량, 브로커리지별 수수료, Binance BNFCR 대체 담보 지원 (commit 9b2a79370)
- **Index**: SPX/VIX 등 비거래 지수
- **CFD**: Contract for Difference

---

## 7. 주문 & 트랜잭션

[Common/Orders/](Common/) + [Engine/TransactionHandlers/](Engine/)

### 주문 타입
`MarketOrder`, `LimitOrder`, `StopMarketOrder`, `StopLimitOrder`, `MarketOnOpenOrder`, `MarketOnCloseOrder`, `ComboMarketOrder`, `ComboLegLimitOrder`, `TrailingStopOrder`.

### 주문 수명주기
```
User.Order() → SubmitOrderRequest → ITransactionHandler
    → FillModel (백테스트) / IBrokerage (라이브)
    → OrderEvent (Submitted/Filled/Canceled/Rejected)
    → algorithm.OnOrderEvent()
```

### 최근 추가된 기능
- Combo 주문 + Python 커스텀 FillModel 지원 (Python/C# 회귀 테스트 포함)
- OptionStrategy 레그 심볼을 생성 시점에 설정 (commit 61b57dc4f)

---

## 8. Brokerages

[Brokerages/](Brokerages/) — `IBrokerage` 인터페이스 구현:
`Connect/Disconnect`, `PlaceOrder/UpdateOrder/CancelOrder`, `GetAccountHoldings/GetCash`, `Subscribe`; 이벤트: `OrderStatusChanged`, `AccountChanged`.

**지원 브로커리지** (2026년 기준):
- US Equities/Options: Interactive Brokers, Alpaca, Charles Schwab, Tradier, TradeStation, Tastytrade
- Forex: OANDA, FXCM
- Crypto: Binance (Spot/Futures/Coin), Coinbase, Bybit (Spot/Futures), Kraken, Bitfinex, dYdX
- 인도: Zerodha, Samco
- 기관: Trading Technologies, Atreyu, Terminal Link
- 시뮬레이션: `PaperBrokerage`, `BacktestingBrokerage`

각 브로커리지는 **별도 GitHub 저장소**(예: `Lean.Brokerages.InteractiveBrokers`)로 관리되고 NuGet 패키지로 LEAN 코어에 플러그인됩니다.

---

## 9. Indicators

[Indicators/](Indicators/) — 170+ 지표. `IIndicator` 패턴: `Update(IBaseData)` / `Reset()` / `IsReady` / `Current`.

- **이동평균**: SMA, EMA, WMA, KAMA, DEMA, TEMA
- **모멘텀**: RSI, MACD, ROC, CCI, Stochastic
- **변동성**: ATR, Bollinger, NATR
- **트렌드**: ADX, Aroon, PSAR
- **거래량**: OBV, VWAP, AD
- **사이클**: Fisher/Hilbert Transform
- **옵션 Greeks**: Delta, Gamma, Vega, Theta, Rho (commit c8934d118에서 `GetSafeTheta`로 decimal overflow 처리)
- **통계**: Beta, Correlation, LogReturn
- **RollingWindow**: O(1) 룩백용 원형 버퍼

### 최근 리팩토링 — SwissArmyKnife
commit f0aefdb51 — Gauss/Butter/HighPass/TwoPoleHighPass/BandPass 다섯 필터를 **서브 인디케이터**로 분리. `.Gauss.Current.Value`처럼 개별 접근 가능.

---

## 10. 유니버스 & 스케줄링

### 유니버스
- **Coarse** — 일 단위 가격/거래량 필터 (전 종목)
- **Fine** — coarse 결과에 펀더멘털 적용
- **Option/Future Chain** — 만기, 행사가, OTM/ITM, 거래량 필터

### 스케줄링 API
```python
self.Schedule.On(
    self.DateRules.EveryDay("SPY"),
    self.TimeRules.AfterMarketOpen("SPY", 30),
    self.Rebalance)
```
`DateRules`: `EveryDay`, `MonthStart`, `WeekEnd`, 커스텀.
`TimeRules`: `At`, `Midnight`, `BeforeMarketClose`, `AfterMarketOpen`.

---

## 11. Optimizer

[Optimizer/](Optimizer/) + [Optimizer.Launcher/](Optimizer.Launcher/)

```python
self.fast = self.GetParameter("fast", 10)
```
전략:
- **Grid Search** — 전수 탐색
- **Random Search** — 랜덤 샘플링
- **Euler Search** — 근사 기울기 기반 (실험적)
- **Genetic** — 클라우드 전용

분산 실행: 클라우드에서 여러 LEAN 인스턴스에 분배, Sharpe/목표 함수 기준 순위.

---

## 12. Research (QuantBook)

[Research/QuantBook.cs](Research/) (~1500 lines) — Jupyter 환경용 QCAlgorithm 쌍둥이.
- `BasicQuantBookTemplate.ipynb`, `KitchenSinkQuantBookTemplate.ipynb`
- `qb.History()`, `qb.AddEquity()`, `qb.SMA()` 등 대부분 API 재사용
- 옵션/선물 체인 히스토리 전용 API 제공

---

## 13. Report / ToolBox / Api

- [Report/](Report/) — 백테스트 HTML 리포트 (Sharpe, Sortino, Calmar, drawdown, 월별 수익률 히트맵, 주문 로그). commit 64b0ef386에서 Analysis 섹션을 Backtest 패킷에 추가.
- [ToolBox/](ToolBox/) — 외부 벤더 데이터 변환기 (AlgoSeek, IQFeed, CME, Kaiko, Coinbase), `RandomDataGenerator`, `FactorFileGenerator`
- [Api/Api.cs](Api/) (~1700 lines) — QuantConnect 클라우드 REST API 클라이언트

---

## 14. Tests — 회귀 테스트 중심

[Tests/](Tests/):
- **RegressionTests.cs** — 500+ 알고리즘을 실제로 백테스트 실행 후 baseline equity curve와 비교
- **AlgorithmRunner** — 테스트 헬퍼
- 각 자산군·기능마다 유닛 테스트 분리

→ **리팩토링할 때 기존 동작이 깨지지 않는다는 강한 보장**이 이 엔진의 프로덕션 급 신뢰성의 핵심입니다.

---

## 15. 배포 (Docker)

- [Dockerfile](Dockerfile) — 메인 LEAN 런타임 (디버거 포함)
- [DockerfileLeanFoundation](DockerfileLeanFoundation) — 베이스 이미지 (Ubuntu + .NET + Python 3.8 + TA-Lib, numpy, pandas, scipy)
- [DockerfileLeanFoundationARM](DockerfileLeanFoundationARM) — ARM64 (M1 Mac, Raspberry Pi)
- [DockerfileJupyter](DockerfileJupyter) — 리서치 환경

---

## 16. 경쟁 프레임워크 비교 (2026 기준)

| | **LEAN** | **Backtrader** | **Zipline** |
|---|---|---|---|
| 언어 | C# 코어 + Python API | 순수 Python | Python 3.5/3.6 위주 |
| 아키텍처 | 이벤트 드리븐, 클라우드 네이티브 | 이벤트 드리븐, 로컬 | 이벤트 드리븐 (느림) |
| 자산군 | 9종 전부 | 주식/선물/Forex/Crypto | 주로 미국 주식 |
| 라이브 트레이딩 | 30+ 브로커리지 기본 내장 | 플러그인 필요 | 미흡 |
| 데이터 | 멀티 자산·대체 데이터 풍부 | BYO 데이터 | Quantopian 번들 |
| 설치 | Docker / .NET SDK | `pip install` | Python 호환성 지옥 |
| 적합 | 프로덕션, 멀티 자산, 클라우드 | 스윙 트레이더, 학습 | 팩터 리서치, 유산 |

→ **LEAN은 "엔드투엔드 통합 플랫폼"**이 필요할 때 독보적입니다. Backtrader는 가볍지만 라이브 배포 인프라는 직접 짜야 하고, Zipline은 사실상 레거시입니다.

---

## 17. 이 리포지터리의 활발한 개발 방향 (최근 커밋)

최근 30개 커밋에서 감지되는 테마:
1. **옵션/전략 심화** — `SwissArmyKnife` 서브 인디케이터 리팩토링, indicator-based 옵션 프라이싱 모델, `OptionStrategy` 레그 심볼 조기 설정
2. **심볼·코포레이트 액션 견고성** — OSI 옵션 티커 (BRK.B 같은 점 포함) 파싱 수정, `SymbolPropertiesDatabaseSymbolMapper` 신규 심볼 대응
3. **데이터 관리** — `DataManager` 잘못된 구독 타입 생성 버그 수정, `AdvanceTime` 타이밍 수정
4. **크립토** — Binance BNFCR 대체 담보, 바이낸스 선물 마진 모델 확장
5. **리포팅** — Analysis를 Backtest 패킷에 추가, results analyzer 도입
6. **알고리즘 상태** — `Idle`, `PendingInput` 새 상태 추가
7. **Python 생태계** — Pythonnet v2.0.53, 파이썬 dict 헤더로 커스텀 SubscriptionDataSource
8. **최신 기능**: Candlestick 차트, Python 커스텀 FillModel + Combo 주문

---

## 18. 신규 입문자를 위한 추천 독서 순서

### Phase 1 — 기반 (1~2일)
1. [Launcher/Program.cs](Launcher/Program.cs) → 실행 시퀀스
2. [Engine/Engine.cs](Engine/Engine.cs) → 오케스트레이션
3. [Engine/AlgorithmManager.cs](Engine/AlgorithmManager.cs) → **이벤트 루프** (LEAN의 심장)
4. [Engine/LeanEngineAlgorithmHandlers.cs](Engine/LeanEngineAlgorithmHandlers.cs) → 핸들러 구성, `Launcher/config.json` 같이 읽기

### Phase 2 — 데이터·주문 (2~3일)
5. [Engine/DataFeeds/FileSystemDataFeed.cs](Engine/) — 백테스트 데이터 흐름
6. [Engine/DataFeeds/Subscription.cs](Engine/) + Enumerators 체인
7. [Common/Orders/Order.cs](Common/) + [Engine/TransactionHandlers/BacktestingTransactionHandler.cs](Engine/)

### Phase 3 — 시큐리티·알고리즘 (2일)
8. [Common/Securities/Security.cs](Common/) + [SecurityPortfolioManager.cs](Common/)
9. [Algorithm/QCAlgorithm.cs](Algorithm/) 및 partial 파일들 훑기
10. 간단한 Algorithm.CSharp 예제 하나를 엔드투엔드로 따라가기

### Phase 4 — 프레임워크·지표 (2일)
11. [Algorithm.Framework/](Algorithm.Framework/) 다섯 모듈
12. [Indicators/Indicator.cs](Indicators/) + 간단한 SMA/RSI 구현 분석
13. [Indicators/SwissArmyKnife.cs](Indicators/) — 최근 리팩토링 패턴 학습

### Phase 5 — 라이브·클라우드 (1~2일)
14. [Brokerages/Brokerage.cs](Brokerages/) + 선호 브로커리지 하나 (예: Binance)
15. [Tests/RegressionTests.cs](Tests/) — 테스트 구조
16. [Report/Report.cs](Report/) + [Api/Api.cs](Api/)

---

## 19. 핵심 원칙 요약

1. **인터페이스 기반 합성** — 모든 것이 교체 가능 (테스트·확장·커스텀 브로커)
2. **결정론적 백테스트** — 같은 입력 → 같은 결과 (회귀 테스트 500+)
3. **구독 기반 데이터** — 알고리즘이 필요한 심볼/해상도만 펄 → 메모리·CPU 효율
4. **모델 드리븐 시장 시뮬레이션** — Fill/Slippage/Fee/Margin/Settlement 모두 분리
5. **TimeSlice 이벤트 루프** — 시간당 한 번, 모든 심볼 데이터를 묶어 알고리즘에 전달
6. **동일 코드 → 백테스트/페이퍼/라이브** — 환경만 바꾸면 그대로 배포

---

**결론**: LEAN은 단순한 백테스터가 아니라 **"퀀트 펀드 수준의 모든 인프라"**(데이터, 리서치, 지표, 주문 집행, 포트폴리오 관리, 리스크, 브로커리지 통합, 리포팅, 최적화)를 코드로 공유하는 프로덕션 플랫폼입니다. 한 번 구조를 익히면 **개인 레벨에서도 헤지펀드 급 인프라를 운영**할 수 있다는 것이 강력한 장점입니다.

---

## 참고 자료 (Sources)

- [GitHub - QuantConnect/Lean](https://github.com/QuantConnect/Lean)
- [LEAN Engine - QuantConnect Docs](https://www.quantconnect.com/docs/v2/lean-engine)
- [LEAN Engine Getting Started](https://www.quantconnect.com/docs/v2/lean-engine/getting-started)
- [QuantConnect Architecture & LEAN Engine Assessment 2026 - AitoCore](https://aitocore.com/en/tool/quantconnect)
- [QuantConnect Review 2026](https://newyorkcityservers.com/blog/quantconnect-review)
- [LEAN Engine Versions](https://www.quantconnect.com/docs/v2/cloud-platform/projects/lean-engine-versions)
- [LEAN Release Notes v15817-v15806](https://www.quantconnect.com/lean/16064/lean-release-notes-v15817-v15806/)
- [LEAN Release Notes v15780-v15766](https://www.quantconnect.com/lean/15939/lean-release-notes-v15780-v15766/)
- [Python Backtesting Landscape 2026](https://python.financial/)
- [Popular Backtesting Tools Comparison - Medium](https://medium.com/@pta.forwork/popular-backtesting-tools-for-algorithmic-trading-a-practical-comparison-and-how-to-use-them-fa09f9fb2480)
- [VectorBT vs Zipline vs Backtrader - Medium](https://medium.com/@trading.dude/battle-tested-backtesters-comparing-vectorbt-zipline-and-backtrader-for-financial-strategy-dee33d33a9e0)
- [Algorithm Framework Overview](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/overview)
- [Alpha Models Key Concepts](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/alpha/key-concepts)
- [Portfolio Construction Key Concepts](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/portfolio-construction/key-concepts)
- [Supported Portfolio Construction Models](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/portfolio-construction/supported-models)
- [QuantConnect Lean.Brokerages.InteractiveBrokers](https://github.com/QuantConnect/Lean.Brokerages.InteractiveBrokers)
