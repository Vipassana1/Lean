# QuantConnect LEAN을 수학과 물리의 언어로 이해하기

LEAN은 한 문장으로 말하면 **금융시장을 실험실 안에 재현하고, 내가 만든 전략이라는 동역학 법칙을 그 위에서 시간에 따라 진화시키는 엔진**입니다.

좀 더 공학적으로 말하면, LEAN은 QuantConnect가 만든 오픈소스 알고리즘 트레이딩 엔진입니다. 백테스트, 리서치, 페이퍼 트레이딩, 라이브 트레이딩을 같은 구조 안에서 실행합니다. C#/.NET으로 만들어진 엔진 코어 위에 Python API도 제공하므로, 사용자는 Python으로 전략을 쓰면서도 내부적으로는 성능 좋은 엔진을 사용하게 됩니다.

이 문서는 LEAN을 단순히 “백테스터”라고 보지 않고, **시간, 상태, 관측, 힘, 제약, 실험 재현성**이라는 수학과 물리의 언어로 풀어 설명합니다.

---

## 1. LEAN은 무엇인가?

수학적으로 보면 트레이딩 전략은 하나의 함수입니다.

```text
전략 f: 시장 상태 x_t -> 행동 a_t
```

여기서 `x_t`는 시각 `t`에서 관측 가능한 시장 상태입니다. 예를 들면 가격, 거래량, 포트폴리오, 주문 상태, 지표값, 현금, 마진, 옵션 Greeks 등이 포함됩니다.

`a_t`는 전략이 내리는 행동입니다. 예를 들면 매수, 매도, 리밸런싱, 주문 취소, 리스크 축소 같은 것들입니다.

물리적으로 보면 LEAN은 다음과 같은 실험 장치입니다.

```text
시장 데이터 = 센서가 읽은 관측값
알고리즘 = 입자 또는 물체의 운동 법칙
포트폴리오 = 현재 계의 상태
주문 = 계에 가하는 힘
브로커리지 = 실제 세계와 연결된 액추에이터
수수료/슬리피지/마진 = 마찰, 저항, 경계조건
백테스트 = 과거 데이터를 이용한 재현 실험
라이브 트레이딩 = 실시간 물리계와의 상호작용
```

즉 LEAN은 “전략을 실행해주는 프로그램”이라기보다, **금융 동역학을 시뮬레이션하고 실제 시장에 연결하는 런타임**에 가깝습니다.

LEAN의 중요한 특징은 다음과 같습니다.

- **오픈소스**: Apache 2.0 라이선스
- **언어 구조**: C# 엔진 코어 + Python 알고리즘 작성 지원
- **자산군**: 주식, 옵션, 선물, 외환, 암호화폐, 지수, CFD 등
- **실행 모드**: 백테스트, 리서치, 페이퍼, 라이브 트레이딩
- **설계 철학**: 모든 주요 부품을 인터페이스 기반으로 교체 가능

---

## 2. 가장 중요한 그림: 시간에 따른 상태 진화

LEAN의 핵심은 시간 루프입니다.

수학적으로는 다음과 같은 이산 시간 동역학으로 볼 수 있습니다.

```text
x_t     = 시각 t의 시장 데이터와 포트폴리오 상태
f       = 사용자가 작성한 알고리즘
a_t     = f(x_t), 즉 주문이나 리밸런싱 같은 행동
g       = 체결, 수수료, 슬리피지, 마진 등을 포함한 시장/브로커 모델
x_{t+1} = g(x_t, a_t)
```

전략은 매 시점마다 시장을 보고 행동을 결정합니다. 그러면 LEAN은 그 행동이 실제로 어떤 결과를 낳는지 계산합니다. 주문이 체결되었는지, 얼마에 체결되었는지, 수수료는 얼마인지, 포트폴리오 가치가 어떻게 바뀌었는지 업데이트합니다.

물리 비유로 말하면:

```text
현재 위치와 속도 x_t를 측정한다
힘 a_t를 가한다
마찰과 제약조건 g를 적용한다
다음 상태 x_{t+1}로 이동한다
```

LEAN의 실행 흐름은 대략 다음과 같습니다.

```text
[Launcher/Program.cs]
   ↓ 설정 파일을 읽고 실행 환경 선택
[Engine/Engine.cs]
   ↓ 데이터, 브로커리지, 결과 처리, 시간 처리 장치 조립
[Engine/AlgorithmManager.cs]
   ↓ TimeSlice 단위로 시간을 전진시키는 메인 루프
[Algorithm/QCAlgorithm.cs 또는 사용자 Python 알고리즘]
   ↓ OnData, OnOrderEvent, 스케줄 콜백 실행
[TransactionHandler / Brokerage]
   ↓ 주문 체결 또는 실제 시장 연결
```

입문자가 가장 먼저 봐야 하는 파일은 다음입니다.

- [Launcher/Program.cs](Launcher/Program.cs): 실행 진입점
- [Engine/Engine.cs](Engine/Engine.cs): 엔진 전체 조립자
- [Engine/AlgorithmManager.cs](Engine/AlgorithmManager.cs): 시간 루프의 중심
- [Algorithm/QCAlgorithm.cs](Algorithm/QCAlgorithm.cs): 사용자가 상속받는 알고리즘 베이스 클래스

---

## 3. TimeSlice: LEAN의 시간 미분 단위

물리에서 연속 시간을 컴퓨터로 계산할 때는 보통 시간을 작은 조각으로 나눕니다.

```text
t = 0, 1, 2, 3, ...
```

LEAN도 비슷합니다. 다만 시간 간격은 초, 분, 일, 틱 등 데이터 해상도에 따라 달라집니다. LEAN은 각 시각에 들어온 여러 심볼의 데이터를 하나로 묶어 `TimeSlice`라는 단위로 알고리즘에 전달합니다.

예를 들어 같은 09:31에 AAPL, MSFT, SPY 데이터가 모두 있으면 LEAN은 이것을 하나의 시간 단면으로 묶습니다.

```text
TimeSlice(09:31)
  ├── AAPL TradeBar
  ├── MSFT TradeBar
  ├── SPY TradeBar
  ├── 주문 이벤트
  ├── 배당/분할 이벤트
  └── 스케줄 이벤트
```

수학적으로는 시장의 상태 벡터를 한 번에 샘플링하는 것과 같습니다.

```text
x_t = [price_AAPL, price_MSFT, price_SPY, holdings, cash, orders, ...]
```

사용자의 `OnData(slice)`는 이 상태 벡터를 받아 행동을 결정하는 함수입니다.

---

## 4. 핸들러 패턴: 실험 장치를 갈아 끼우는 구조

LEAN의 확장성은 핸들러 패턴에서 나옵니다.

핸들러는 물리 실험 장치의 부품과 비슷합니다. 같은 실험 테이블 위에서 센서, 기록 장치, 액추에이터, 시뮬레이터를 갈아 끼울 수 있는 구조입니다.

| 역할 | 수학/물리적 의미 | 백테스트 | 라이브 |
|---|---|---|---|
| `IDataFeed` | 외부 세계를 관측하는 센서 | 파일 데이터 읽기 | 실시간 데이터 수신 |
| `ITransactionHandler` | 행동을 상태 변화로 바꾸는 장치 | 체결 모델로 시뮬레이션 | 브로커에 주문 전송 |
| `IResultHandler` | 실험 결과 기록계 | 백테스트 결과 저장 | 실시간 결과 전송 |
| `IRealTimeHandler` | 시간 기준계 | 과거 시간 재생 | 실제 시계 사용 |
| `ISetupHandler` | 초기조건 설정 | 백테스트 초기화 | 계좌/브로커 연결 |
| `IHistoryProvider` | 과거 관측값 제공 | 디스크에서 읽기 | 브로커/API에서 조회 |

이 구조의 장점은 큽니다.

```text
같은 알고리즘 f
다른 환경 g
다른 실행 결과
```

즉 사용자는 전략 코드를 거의 바꾸지 않고도 백테스트에서 라이브로 옮겨갈 수 있습니다. 물리로 치면, 같은 운동 법칙을 진공 실험과 마찰 있는 실험에 모두 적용할 수 있게 만든 것입니다.

---

## 5. QCAlgorithm: 전략을 쓰기 위한 좌표계

사용자는 보통 `QCAlgorithm`을 상속해서 전략을 작성합니다.

Python으로 쓰면 대략 이런 모양입니다.

```python
class MyAlgorithm(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020, 1, 1)
        self.SetCash(100000)
        self.symbol = self.AddEquity("SPY").Symbol

    def OnData(self, data):
        if not self.Portfolio[self.symbol].Invested:
            self.SetHoldings(self.symbol, 1.0)
```

수학적으로 보면:

```text
Initialize() = 초기조건 설정
OnData()     = 시간 t마다 호출되는 상태 전이 규칙
Portfolio    = 현재 상태 변수
Order()      = 상태에 가하는 제어 입력
```

물리적으로 보면:

```text
Initialize()는 실험 시작 전 장치 세팅
OnData()는 매 시간마다 계를 관측하고 힘을 가하는 법칙
Portfolio는 현재 에너지/위치/운동량 같은 상태
Order는 계에 외부 힘을 가하는 행위
```

`QCAlgorithm`은 많은 기능을 제공합니다.

- 주문: `MarketOrder`, `LimitOrder`, `SetHoldings`
- 데이터 추가: `AddEquity`, `AddOption`, `AddFuture`, `AddCrypto`
- 과거 데이터: `History`
- 지표: `SMA`, `EMA`, `RSI`, `BB`
- 스케줄링: 매일 장 시작 후 30분, 월초, 주말 등
- 포트폴리오와 현금 관리
- 리스크와 주문 이벤트 처리

---

## 6. Algorithm Framework: 전략을 다섯 개의 연산자로 분해하기

LEAN에는 전략을 모듈식으로 쓰는 Algorithm Framework가 있습니다.

전체 전략을 하나의 거대한 함수로 쓰는 대신, 다음 다섯 단계로 나눕니다.

```text
Universe Selection → Alpha → Portfolio Construction → Risk Management → Execution
```

수학적으로는 함수 합성입니다.

```text
Orders
= Execution(
    Risk(
      Portfolio(
        Alpha(
          Universe(market)
        )
      )
    )
  )
```

각 단계의 의미는 다음과 같습니다.

| 모듈 | 질문 | 출력 |
|---|---|---|
| Universe Selection | 어떤 자산을 볼 것인가? | Symbol 목록 |
| Alpha | 어떤 방향의 예측이 있는가? | Insight |
| Portfolio Construction | 얼마나 들고 갈 것인가? | 목표 비중 |
| Risk Management | 위험이 너무 크지 않은가? | 조정된 목표 비중 |
| Execution | 실제 주문은 어떻게 낼 것인가? | 주문 |

물리 비유로 보면:

```text
Universe = 실험 대상 입자 선택
Alpha = 힘의 방향 예측
Portfolio = 각 입자에 배분할 질량/에너지 결정
Risk = 폭주하지 않도록 제약조건 적용
Execution = 실제로 힘을 가하는 방식
```

이 구조는 큰 전략을 다룰 때 특히 좋습니다. 한 파일에 모든 판단을 몰아넣지 않고, 예측, 포트폴리오, 리스크, 체결을 서로 다른 연산자로 분리할 수 있기 때문입니다.

---

## 7. 데이터 계층: 시장을 관측 가능한 신호로 바꾸기

LEAN에서 데이터는 단순한 CSV가 아닙니다. 모든 데이터는 시간축 위에 놓인 관측값입니다.

대표적인 데이터 타입은 다음과 같습니다.

- `TradeBar`: 일정 시간 동안의 OHLCV
- `QuoteBar`: Bid/Ask 기준 OHLC
- `Tick`: 개별 틱 데이터
- `OptionChain`: 특정 기초자산의 옵션 체인
- `FutureChain`: 선물 체인
- `Fundamental`: 기업 재무/펀더멘털 데이터
- 사용자 정의 `BaseData`

수학적으로는 데이터가 모두 시간의 함수입니다.

```text
price: t -> R
volume: t -> R
option_chain: t -> {contracts}
fundamental: t -> feature vector
```

LEAN의 데이터 구독은 “필요한 센서만 켠다”는 뜻입니다.

```python
self.AddEquity("SPY", Resolution.Minute)
```

이 코드는 “SPY의 분봉 센서를 켜라”는 뜻입니다. LEAN은 사용자가 구독한 심볼과 해상도의 데이터만 읽고 정렬합니다.

데이터 처리 과정에는 여러 보정 장치가 들어갑니다.

- `FillForwardEnumerator`: 데이터가 비어 있을 때 이전 값을 이어붙임
- `SortEnumerator`: 시간 순서 정렬
- `MappingEventProvider`: 티커 변경 처리
- `SplitEventProvider`: 주식 분할 처리
- `DividendEventProvider`: 배당 처리
- `PriceScaleFactorEnumerator`: 가격 조정 계수 적용

물리 실험으로 치면, 센서 노이즈와 단위 변환, 좌표계 보정을 처리하는 단계입니다.

---

## 8. 백테스트의 본질: 결정론적 재현 실험

백테스트는 과거 데이터를 이용한 실험입니다.

중요한 점은 결정론입니다.

```text
같은 데이터 + 같은 코드 + 같은 설정 = 같은 결과
```

이것은 수학의 함수 개념과 같습니다. 입력이 같으면 출력이 같아야 합니다.

```text
BacktestResult = F(strategy, data, config)
```

좋은 백테스트 엔진은 이 함수가 흔들리지 않게 만들어야 합니다. LEAN은 회귀 테스트를 많이 갖고 있어서, 엔진 내부 구현을 바꿔도 기존 전략 결과가 갑자기 바뀌지 않도록 확인합니다.

물리로 비유하면, 같은 초기조건에서 같은 실험을 반복했을 때 같은 궤적이 나와야 한다는 뜻입니다. 그래야 전략이 좋아서 수익이 난 것인지, 실험 장치가 우연히 흔들린 것인지 구분할 수 있습니다.

---

## 9. 주문과 체결: 힘을 가했을 때 실제로 일어나는 일

전략이 주문을 내는 것은 계에 힘을 가하는 것과 비슷합니다.

```text
Order = 제어 입력 u_t
```

하지만 현실에서는 “매수하라”고 말한다고 원하는 가격에 바로 체결되지 않습니다. 시장에는 마찰이 있습니다.

LEAN은 이 마찰을 여러 모델로 분리합니다.

| 모델 | 물리적 비유 | 의미 |
|---|---|---|
| `FillModel` | 힘이 실제 운동으로 바뀌는 방식 | 주문 체결 여부와 가격 |
| `SlippageModel` | 마찰/미끄러짐 | 기대 가격과 실제 체결 가격 차이 |
| `FeeModel` | 에너지 손실 | 수수료 |
| `BuyingPowerModel` | 허용 가능한 힘의 크기 | 매수 가능 금액, 레버리지 |
| `MarginModel` | 경계조건 | 증거금, 강제청산 조건 |
| `SettlementModel` | 시간 지연 | 결제일 처리 |

주문 흐름은 다음과 같습니다.

```text
User.Order()
  → SubmitOrderRequest
  → TransactionHandler
  → FillModel 또는 Brokerage
  → OrderEvent
  → OnOrderEvent()
```

백테스트에서는 `FillModel`이 시장 체결을 시뮬레이션합니다. 라이브에서는 실제 브로커리지 API가 주문을 처리합니다.

---

## 10. 포트폴리오: 상태 벡터와 보존량

포트폴리오는 전략의 현재 상태입니다.

수학적으로는 상태 벡터입니다.

```text
P_t = [cash, holdings, average_price, unrealized_pnl, margin, fees, ...]
```

또는 자산별 보유량 벡터로 볼 수도 있습니다.

```text
h_t = [shares_AAPL, shares_MSFT, contracts_ES, BTC_amount, ...]
```

포트폴리오 가치는 가격 벡터와 보유량 벡터의 내적에 가깝습니다.

```text
PortfolioValue_t = cash_t + p_t · h_t
```

여기서 `p_t`는 가격 벡터, `h_t`는 보유량 벡터입니다.

물리적으로는 계의 총 에너지와 비슷합니다. 매수, 매도, 가격 변화, 수수료, 환율 변화가 모두 이 에너지를 바꿉니다.

LEAN의 주요 포트폴리오 구성요소는 다음입니다.

- `Security`: 각 자산의 상태
- `SecurityHolding`: 보유 수량, 평균 단가, 평가손익
- `SecurityPortfolioManager`: 전체 포트폴리오 가치, 현금, 마진, 손익 관리
- `CashBook`: 여러 통화와 환율 관리

---

## 11. 지표: 시장 신호에 작용하는 필터

기술적 지표는 가격 신호에 적용하는 수학적 필터입니다.

예를 들어 이동평균은 고주파 노이즈를 줄이는 저역통과 필터처럼 볼 수 있습니다.

```text
SMA_t = (p_t + p_{t-1} + ... + p_{t-n+1}) / n
```

EMA는 최근 데이터에 더 큰 가중치를 주는 재귀 필터입니다.

```text
EMA_t = α p_t + (1 - α) EMA_{t-1}
```

RSI, MACD, Bollinger Bands, ATR 같은 지표도 모두 가격과 거래량이라는 신호를 변환하는 연산자입니다.

```text
indicator: price history -> signal
```

LEAN의 지표는 대체로 다음 인터페이스를 따릅니다.

- `Update`: 새 데이터를 넣는다
- `Current`: 현재 지표값을 읽는다
- `IsReady`: 충분한 데이터가 쌓였는지 확인한다
- `Reset`: 초기화한다

물리적으로 말하면, 센서에서 들어온 원시 신호를 필터와 계측기로 통과시켜 더 해석하기 쉬운 신호로 바꾸는 과정입니다.

---

## 12. 옵션과 선물: 상태공간이 더 큰 세계

주식은 비교적 단순합니다.

```text
가격 p_t
보유량 h_t
```

옵션은 상태공간이 더 큽니다.

```text
option_value = f(underlying_price, strike, time_to_expiry, volatility, rate, ...)
```

옵션의 Greeks는 이 함수의 민감도입니다.

```text
Delta = ∂V/∂S
Gamma = ∂²V/∂S²
Theta = ∂V/∂t
Vega  = ∂V/∂σ
Rho   = ∂V/∂r
```

즉 Greeks는 옵션 가격이라는 곡면의 기울기와 곡률입니다.

선물은 만기와 롤오버가 중요합니다. 연속 선물은 여러 만기 계약을 하나의 긴 시간축으로 이어 붙인 좌표계라고 볼 수 있습니다.

LEAN이 여러 자산군을 지원한다는 말은, 각 자산마다 다른 상태공간과 제약조건을 엔진 안에 갖고 있다는 뜻입니다.

---

## 13. 리서치 환경 QuantBook: 실험 전 칠판 계산

`QuantBook`은 Jupyter 환경에서 쓰는 리서치 도구입니다.

전략을 바로 엔진에 올리기 전에, 데이터를 불러오고, 가설을 확인하고, 지표를 계산하고, 시각화하는 데 사용합니다.

물리로 말하면 본 실험 전에 칠판과 노트북에서 하는 예비 계산입니다.

```text
QuantBook = 가설 탐색
QCAlgorithm = 시간 루프 안에서 실제 전략 실행
```

둘은 많은 API를 공유합니다. 그래서 리서치에서 검토한 아이디어를 전략 코드로 옮기기 쉽습니다.

---

## 14. Optimizer: 파라미터 공간 탐색

전략에는 보통 파라미터가 있습니다.

```text
fast = 10
slow = 30
threshold = 0.02
```

수학적으로는 전략 성과가 파라미터 벡터의 함수가 됩니다.

```text
score = J(θ)
θ = [fast, slow, threshold, ...]
```

Optimizer는 이 함수의 좋은 지점을 찾습니다.

- Grid Search: 격자 위의 모든 점을 확인
- Random Search: 무작위 샘플링
- Euler Search: 기울기 비슷한 정보를 이용한 탐색
- Genetic: 진화 알고리즘 방식의 탐색

물리로 비유하면 에너지 지형 위에서 낮은 골짜기나 높은 봉우리를 찾는 문제와 비슷합니다. 다만 금융에서는 과최적화라는 함정이 큽니다. 과거 데이터에서만 아름다운 해를 찾고 미래에는 무너질 수 있기 때문입니다.

---

## 15. 백테스트, 페이퍼, 라이브의 차이

세 모드는 같은 전략을 서로 다른 세계에 넣는 것입니다.

| 모드 | 세계 | 시간 | 체결 |
|---|---|---|---|
| 백테스트 | 과거 데이터 | 재생되는 시간 | 모델이 시뮬레이션 |
| 페이퍼 | 실시간 시장 | 실제 시간 | 가상 체결 |
| 라이브 | 실시간 시장 | 실제 시간 | 실제 브로커 체결 |

수학적으로는 전략 함수 `f`는 같고, 외부 환경 `g`가 달라집니다.

```text
x_{t+1} = g(x_t, f(x_t))
```

백테스트의 `g`는 파일 데이터와 체결 모델입니다. 라이브의 `g`는 실제 데이터 피드와 브로커리지입니다.

LEAN의 강점은 이 세 세계를 같은 코드 구조로 다룬다는 점입니다.

---

## 16. Docker와 배포: 같은 실험실을 어디서나 재현하기

복잡한 실험은 환경이 달라지면 결과가 달라집니다. Python 버전, .NET 버전, 라이브러리, OS가 조금만 달라도 문제가 생길 수 있습니다.

Docker는 실험실 전체를 컨테이너로 포장하는 방법입니다.

```text
Docker image = 고정된 실험실
Docker container = 실행 중인 실험실
```

LEAN은 Docker 기반 실행을 지원하므로, 로컬, 서버, 클라우드에서 비슷한 환경을 재현할 수 있습니다.

---

## 17. 다른 백테스팅 프레임워크와 비교

LEAN을 Backtrader나 Zipline 같은 도구와 비교하면 차이가 분명합니다.

| 항목 | LEAN | Backtrader | Zipline |
|---|---|---|---|
| 성격 | 엔드투엔드 트레이딩 엔진 | 가벼운 Python 백테스터 | 팩터 리서치 중심 레거시 |
| 언어 | C# 코어 + Python API | Python | Python |
| 강점 | 멀티 자산, 라이브, 브로커리지, 클라우드 | 단순함, 학습 용이 | Quantopian식 연구 구조 |
| 약점 | 구조가 크고 학습량이 많음 | 라이브 인프라는 직접 구성 필요 | 현대 환경에서 유지보수 부담 |

물리 비유로 말하면:

```text
Backtrader = 개인 실험 키트
Zipline = 특정 실험을 위한 오래된 장비
LEAN = 센서, 제어기, 기록계, 실제 장비 연결까지 갖춘 종합 실험실
```

---

## 18. LEAN을 읽는 추천 순서

처음부터 모든 파일을 보려고 하면 어렵습니다. 다음 순서로 보면 구조가 잘 잡힙니다.

### 1단계: 시간 루프 이해

1. [Launcher/Program.cs](Launcher/Program.cs)
2. [Engine/Engine.cs](Engine/Engine.cs)
3. [Engine/AlgorithmManager.cs](Engine/AlgorithmManager.cs)

목표는 “LEAN이 어떻게 시작되고, 어떻게 시간을 전진시키며, 언제 내 알고리즘을 호출하는가”를 이해하는 것입니다.

### 2단계: 사용자 전략 이해

1. [Algorithm/QCAlgorithm.cs](Algorithm/QCAlgorithm.cs)
2. `QCAlgorithm.Trading.cs`
3. `QCAlgorithm.History.cs`
4. `QCAlgorithm.Indicators.cs`

목표는 “내가 Python/C#에서 부르는 API가 엔진 내부 어디로 이어지는가”를 보는 것입니다.

### 3단계: 데이터 흐름 이해

1. [Engine/DataFeeds/](Engine/DataFeeds/)
2. `Subscription`
3. `Synchronizer`
4. Enumerator 체인

목표는 “데이터가 파일 또는 실시간 피드에서 어떻게 `OnData`까지 오는가”입니다.

### 4단계: 주문과 포트폴리오 이해

1. [Common/Orders/](Common/Orders/)
2. [Engine/TransactionHandlers/](Engine/TransactionHandlers/)
3. [Common/Securities/](Common/Securities/)

목표는 “내 주문이 어떻게 체결 이벤트와 포트폴리오 변화로 바뀌는가”입니다.

### 5단계: 프레임워크와 지표 이해

1. [Algorithm.Framework/](Algorithm.Framework/)
2. [Indicators/](Indicators/)
3. [Tests/](Tests/)

목표는 “큰 전략을 어떻게 모듈로 쪼개고, 엔진이 그 동작을 어떻게 검증하는가”입니다.

---

## 19. 핵심 원칙 요약

LEAN을 이해할 때는 다음 여섯 가지만 기억해도 큰 그림이 잡힙니다.

1. **전략은 함수다**

   ```text
   시장 상태 -> 투자 행동
   ```

2. **백테스트는 결정론적 실험이다**

   ```text
   같은 입력 -> 같은 결과
   ```

3. **포트폴리오는 상태 벡터다**

   ```text
   현금, 보유량, 평가손익, 마진, 주문 상태
   ```

4. **지표는 신호 필터다**

   ```text
   가격/거래량 신호 -> 해석 가능한 특징값
   ```

5. **주문은 제어 입력이고, 체결 모델은 마찰이다**

   ```text
   의도한 행동과 실제 결과 사이에는 수수료, 슬리피지, 유동성 제약이 있다
   ```

6. **핸들러는 실험 장치의 교체 가능한 부품이다**

   ```text
   파일 데이터, 실시간 데이터, 백테스트 체결, 실제 브로커 체결을 같은 구조로 다룬다
   ```

---

## 결론

LEAN은 단순한 백테스터가 아닙니다.

수학의 언어로 말하면, LEAN은 **시장 상태공간 위에서 전략 함수의 시간 진화를 계산하는 엔진**입니다.

물리의 언어로 말하면, LEAN은 **금융시장을 실험실 안에 재현하고, 전략이라는 운동 법칙을 다양한 마찰과 제약조건 아래에서 시험하는 장치**입니다.

그리고 실전의 언어로 말하면, LEAN은 데이터 수집, 지표 계산, 포트폴리오 관리, 주문 체결, 리스크 관리, 리포팅, 최적화, 브로커리지 연결까지 한 번에 제공하는 **프로덕션급 알고리즘 트레이딩 플랫폼**입니다.

처음에는 구조가 커 보이지만, 핵심은 단순합니다.

```text
시간이 흐른다
데이터가 들어온다
전략이 판단한다
주문이 나간다
포트폴리오가 바뀐다
결과가 기록된다
```

이 루프를 이해하면 LEAN 전체가 하나의 거대한 물리 시뮬레이터처럼 보이기 시작합니다.

---

## 참고 자료

- [GitHub - QuantConnect/Lean](https://github.com/QuantConnect/Lean)
- [LEAN Engine - QuantConnect Docs](https://www.quantconnect.com/docs/v2/lean-engine)
- [LEAN Engine Getting Started](https://www.quantconnect.com/docs/v2/lean-engine/getting-started)
- [Algorithm Framework Overview](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/overview)
- [Alpha Models Key Concepts](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/alpha/key-concepts)
- [Portfolio Construction Key Concepts](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/portfolio-construction/key-concepts)
