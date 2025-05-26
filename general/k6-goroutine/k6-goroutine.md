# K6를 통해 배우는 Goroutine과 Multi-threading의 차이

---

## 1. K6란?

K6는 Grafana사에서 **Go 언어로 개발한 부하 테스트 및 성능 테스트 도구**로, 실제 사용 환경을 시뮬레이션하는 데 최적화되어 있습니다.

* 테스트 시나리오는 JavaScript로 작성하며, CLI를 통해 실행합니다.
* 수천에서 수십만 개의 Virtual User(VU)를 고루틴으로 시뮬레이션할 수 있습니다.
* 내부적으로는 JS 런타임으로 `goja`를 사용합니다.

---

### 구조 개요

```text
+------------------+
|   CLI Entry      | --> main.go
+------------------+
         |
         v
+-------------------------+
|     Executor & VU Pool  | --> lib/executor/
+-------------------------+
         |
         v
+----------------------------+
| JS Runtime (goja Runner)   | --> js/
+----------------------------+
         |
         v
+----------------------+
| lib/http, lib/ws, ...|
+----------------------+
```

### ❓고루틴 생성 이전에 JS 런타임을 단 한 번 실행할 수는 없는가?

* 각 VU 고루틴은 **자체적으로 JS 런타임 인스턴스**를 생성하여 코드를 실행합니다. 이를 통해 **독립적인 실행 환경**을 제공하고, 상태 격리를 달성할 수 있습니다.
* 각 고루틴이 개별적인 JS 런타임 인스턴스를 가지더라도, Go의 **가비지 컬렉터(GC)** 가 메모리 관리와 효율성을 보장하기 때문에 메모리 리소스에 큰 문제가 발생하지 않습니다.
* 결과적으로, **각 VU 고루틴**은 **시나리오를 파싱한 후의 데이터를 고유하게 관리**하며 실행합니다.

### 예시 시나리오와 결과

```js
// 예시 시나리오
// 10명의 VU를 30초 동안 HTTP GET 요청을 보내고,
// 응답 상태가 200인지 확인 후 1초씩 대기하는 시나리오
import http from 'k6/http';
import { sleep, check } from 'k6';

export let options = {
  vus: 10,
  duration: '30s',
};

export default function () {
  let res = http.get('https://test-api.example.com');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  sleep(1);
}
```

```bash
// 실행
k6 run script.js
```

```text
// 예시 결과
running (30.0s), 10/10 VUs, 300 complete and 0 interrupted iterations
default ✓ [======================================] 10 VUs  30s

     data_received........: 1.2 MB  40 kB/s
     data_sent............: 0.5 MB  17 kB/s
     http_req_duration....: avg=120ms min=100ms max=150ms p(95)=140ms p(99)=145ms
     http_reqs............: 300    10.0/s
     iteration_duration...: avg=1.12s min=1.1s  max=1.2s
     iterations...........: 300    10.0/s
     vus..................: 10     min=10  max=10
     vus_max..............: 10     min=10  max=10
```

### 주요 기능 요약

| 기능            | 설명                                |
| ------------- | --------------------------------- |
| HTTP 요청 지원    | GET, POST, PUT, DELETE 등          |
| WebSocket 테스트 | 실시간 서비스 테스트 가능                    |
| Thresholds    | SLA (Service Level Agreement) 조건 정의 가능                      |
| Checks        | Assertion 기능 제공                   |
| Metrics 출력    | JSON, InfluxDB, Prometheus 등으로 출력 |
| Cloud 실행      | Grafana Cloud 확장 가능               |

---

## 2. K6 내부 구조 및 폴더 설명 ( v1.0.0 기준 )

```text
k6/
├── cmd/           # CLI 명령 처리
├── js/            # JavaScript 런타임 연동 (goja)
├── lib/           # HTTP, WebSocket 등 라이브러리 모듈
├── metrics/       # 메트릭 수집 및 처리
├── internal/      # 내부 구현 세부사항
└── main.go        # 프로그램 진입점
```

진입점:

```go
package main

import (
    "go.k6.io/k6/cmd"
)

func main() {
    cmd.Execute()
}
```

---

## 3. 시뮬레이션 플로우 및 내부 동작 구조

### 1) JavaScript 스크립트 파싱

* 위치: `js/modules/`
* **goja 엔진**을 통해 JavaScript를 파싱하고 실행합니다.
* JS의 함수(`http.get`, `sleep`, `check`)는 Go로 구현된 모듈과 연결됩니다.

### 2) VU(Virtual User) 생성 및 실행

* 위치: `lib/executor/`
* 각 VU는 **고루틴**으로 실행되며, 루프 안에서 JS를 반복 실행합니다.
* 각 실행은 `vu.Run()`을 통해 수행됩니다.

```go
func (e *Executor) Run() {
  for i := 0; i < e.options.VUs; i++ {
    vu := NewVU(i)
    go vu.Run()
  }
}
```

### 3) Metrics 수집

* 위치: `metrics/`
* 각 Virtual User(VU)는 테스트 실행 중 수집한 데이터를 Sample 객체로 생성하여, 이를 채널을 통해 전달합니다. 이러한 구조는 고루틴 간의 비동기 통신을 가능하게 하며, 수집된 메트릭을 효율적으로 처리할 수 있도록 합니다.

📌 Sample은 특정 시점의 단일 메트릭 측정값을 나타내며, 메트릭 이름, 값, 타임스탬프, 태그 등의 정보를 포함합니다.

```go
// k6/metrics/sample.go
type Sample struct {
    TimeSeries TimeSeries // 메트릭 이름과 태그 집합을 포함합니다.
    Time       time.Time // 샘플이 기록된 시간입니다.
    Value      float64 // 측정된 메트릭 값입니다.
    Metadata   map[string]string //고유한 메타데이터를 포함할 수 있습니다.
}
```

📌 SampleContainer는 샘플을 수집하는 인터페이스로, GetSamples() 메서드를 통해 샘플들을 반환합니다.

```go
type SampleContainer interface {
    GetSamples() []Sample
}
```
이를 통해 다양한 형태의 샘플 수집이 가능하며, 샘플들을 그룹화하거나 추가 정보를 첨부할 수 있습니다.

📌 ConnectedSampleContainer는 SampleContainer를 확장한 인터페이스로, 시간 및 태그가 동일한 샘플들을 그룹화할 때 사용됩니다.

```go
type ConnectedSampleContainer interface {
    SampleContainer
    GetTags() *TagSet
    GetTime() time.Time
}
```
이 구조는 동일한 시간과 태그를 가진 샘플들을 효율적으로 처리할 수 있도록 도와줍니다.

📌 메트릭 수집 흐름 예시
아래는 k6에서 메트릭을 수집하고 처리하는 흐름을 간단히 나타낸 예시입니다:

```go
// 메트릭 생성
metric := stats.New("my_metric", stats.Counter)

// 샘플 생성
sample := stats.Sample{
    TimeSeries: stats.TimeSeries{
        Metric: metric,
        Tags:   stats.NewSampleTags(map[string]string{"endpoint": "/api"}),
    },
    Time:  time.Now(),
    Value: 1,
}

// 샘플을 채널을 통해 전송
sampleChannel <- sample
```
이러한 방식으로 각 VU는 샘플을 생성하고, 이를 채널을 통해 메트릭 수집기(Sink)로 전달합니다.

📌 Sink는 메트릭 데이터를 수집하고 처리하는 구조체입니다. 각 메트릭 유형(Counter, Gauge, Rate, Trend)에 대해 별도의 Sink가 존재하며, 이들은 Add() 메서드를 통해 Sample을 수신하고, Format() 메서드를 통해 결과를 출력 형식으로 변환합니다.

예를 들어, CounterSink는 다음과 같이 정의되어 있습니다:

```go
// k6/metrics/sink.go
type CounterSink struct {
    Value float64
    First time.Time
}

func (c *CounterSink) Add(s Sample) {
    c.Value += s.Value
    if c.First.IsZero() {
        c.First = s.Time
    }
}
```

---

## 4. 병렬성과 확장성 설계

### 구조 요약

```text
Main Engine
   |
   v
VU 고루틴 Pool (go 루틴)
   |   |   |
RunOnce → Sample → 채널 → Stats Engine → CLI/DB 출력
```

* VU는 고루틴으로 실행되며, 경량 스레드이므로 수천 개의 고루틴도 문제 없이 실행할 수 있습니다.
* 통계는 채널을 통해 전달되며, Lock-free 방식으로 통신됩니다. 처리 루틴은 단일 고루틴에서 수행됩니다.

* **Go의 Lock-free 방식** 이란?
     * 병렬 처리 중 성능을 개선하고 데드락을 피하기 위해, 채널, Compare-And-Swap 등을 활용하는 방식
     * 특히, 채널 방식은 내부적으로는 lock를 사용할 수 있지만, 사용자 관점에서 lock 없이 데이터를 안전하게 주고 받을 수 있습니다.

---

## 5. 결론: 성능을 위해 설계된 K6

### ✅ 고루틴 기반 병렬성

* 각 VU는 고루틴으로 실행되어 **가볍고 뛰어난 병렬성**을 제공합니다.
* Go의 스케줄러는 수천 개의 고루틴을 효율적으로 관리할 수 있습니다.

### ✅ 비동기 채널로 Metric 전송

* VU는 Sample을 생성한 후 즉시 채널로 전송합니다.
* 통계 집계는 별도의 루틴에서 처리되며, 병목 현상이 발생하지 않습니다.

### ✅ Stats Engine은 단일 루틴 처리

* Aggregator가 모든 Sample을 처리합니다.
* 락 없이 버퍼에 집계한 후 주기적으로 Export합니다.

### 🎯 실전 병목 회피 전략

| 전략                 | 설명                                         |
| ------------------- | -------------------------------------------- |
| 채널 버퍼 충분 확보       | VU가 채널 전송 시 대기하지 않도록 채널 버퍼를 충분히 확보합니다. |
| Sample 최소화          | 구조체 크기를 줄여 메모리 효율성을 높입니다. |
| Aggregator 최적화    | 버퍼에 먼저 집계한 후 일괄적으로 Export합니다. |

---

## 6. 핵심 문장 정리

> "k6는 부하 테스트 도구 스스로가 병목이 되지 않도록, 고루틴 기반 병렬성과 채널 기반 비동기 통계를 설계하여, 테스트 대상의 성능을 왜곡하지 않는 구조를 갖추고 있습니다."

---

## 7. Go의 고루틴이 쓰레드보다 유리한 이유

Go는 동시성 처리에서 고루틴(Goroutine)이라는 가벼운 단위를 사용합니다. 이는 전통적인 스레드보다 훨씬 효율적입니다.

---

### 고루틴의 장점

* **User-level 스레드**: OS의 스레드가 아닌 Go 런타임에서 관리하는 스레드이므로, 환경 제약이 적으며 문맥 전환이 빠릅니다.
* **낮은 환경 의존성**: 스레드는 OS 자원에 민감하지만, 고루틴은 사용자 공간에서 실행되어 환경 제약이 적습니다.
* **간단한 동시성 구현**: 문맥 전환(Context Switch)이 사용자 공간에서 빠르게 일어납니다.
* **메모리 효율성**: 고루틴은 약 2KB의 스택으로 시작하지만, 스레드는 보통 256KB 이상을 사용합니다.
* **비동기 I/O 처리**: 고루틴은 논블로킹 I/O 기반으로 작동해, 작업이 막히지 않고 효율적입니다.

---

### GMP 모델

<image src="./gmp-model.png" width="500px">

* Go는 G(Goroutine)를 P(Processor)가 스케줄링하고, M(Machine, OS 스레드)에 할당하여 실행합니다.
* 이 과정이 사용자 공간에서 이루어지기 때문에 문맥 전환이 매우 빠릅니다.

---

### 병렬성의 한계

고루틴을 수천 개 생성할 수는 있지만, 실제로 동시에 실행되는 수는 **CPU의 코어 수(P)**에 따라 제한됩니다.  
예를 들어, 4코어 CPU에서 1000개의 VU를 설정하더라도, 동시에 병렬 실행되는 수는 최대 4개뿐이며, 나머지는 순서를 기다리게 됩니다.

그러나, Go언어의 효율적인 메모리 사용과 빠른 문맥 전환을 통해 쓰레드 방식보다 더 많은 작업을 경량으로 처리할 수 있습니다.

---

### 비유

**스레드는 버스를 몰아야 하는 기사**,  
**고루틴은 자전거를 타는 사람들**입니다.  

버스는 크고 무겁고 관리가 복잡하지만, 자전거는 가볍고 수백 명이 동시에 움직이기 좋습니다.

---

## 8. 참고 코드 위치 (K6 GitHub)

| 기능         | 파일 위치                                    |
| ---------- | ---------------------------------------- |
| VU 실행 및 루프 | `lib/executor/vu_handle.go`<br> vuHandle.runLoopsIfPossible() 함수에서 VU의 반복 실행 로직을 처리합니다.        |
| Sample 생성  | `lib/netext/httpext/transport.go`<br> HTTP 요청의 RoundTrip 과정에서 메트릭 샘플을 생성합니다.                |
| Sample 전송  | `metrics/sample.go`<br> Sample 구조체와 관련된 메트릭 샘플의 정의와 처리를 담당합니다.                 |
| Stats 처리   | `internal/metrics/engine/`<br> 테스트 실행 중 수집된 메트릭을 집계하고 처리하는 로직이 포함되어 있습니다. |

---

## 📌 K6를 활용하면 좋은 케이스 정리

`bombardier`, `vegeta`는 모두 **HTTP 부하 테스트 도구**로,
웹 서버의 성능을 측정하거나 특정 API의 응답 처리 능력을 확인하는 데 유용합니다.

그러나 이들은 주로 **단순한 API 부하 테스트**에 적합하며,
**RPS(rate per second)** 및 **동시 연결 수** 조절을 통해 빠르게 부하를 발생시키는 데 특화되어 있습니다.

---

## ✅ K6가 특히 강점을 가지는 상황

### 🧠 **복잡한 사용자 시나리오 테스트**
- JWT 로그인 → 인증 토큰 저장 → 인증 API 호출
- 조건부 흐름 테스트 (예: 로그인 실패 시 재로그인)
- 다양한 API를 조합한 순차 실행

> `bombardier`, `vegeta`는 단일 URL 테스트에 강하지만, 이런 흐름은 지원하지 않음.

---

### 📜 **스크립트 기반 시나리오 구성**
- JavaScript 기반 시나리오 작성 가능
- `if`, `try/catch`, `loop` 등 조건 제어 및 로직 구현 가능
- 실제 사용자 동작을 코드 수준에서 유연하게 작성

---

### 📊 **성능 모니터링 도구와의 통합**
- Grafana, InfluxDB, Prometheus 등과 통합 가능
- 실시간 대시보드 제공
- 성능 지표(p95, 실패율, VU별 응답 등)를 시각화 가능

---

### 👥 **Virtual Users (VUs) 기반 테스트**
- **Virtual Users (가상 사용자)** 개념으로 실제 유저 동작을 시뮬레이션
- `bombardier`, `vegeta`에는 없는 기능
- 로그인/세션/인증 상태 유지하면서 테스트 가능

---

### 🚀 **분산 테스트 지원**
- 단일 머신의 CPU, 메모리, 네트워크 한계로 인해 고부하 테스트가 제한적
- K6는 여러 머신에서 동시에 실행 가능 → **수십만 RPS 부하도 시뮬레이션 가능**
- K6 Cloud 또는 K6 Operator(Kubernetes)로 쉽게 분산 구성 가능

---

### 🧪 **WebSocket 테스트**
- WebSocket 또는 gRPC와 같은 실시간 프로토콜 테스트 일부 지원
- HTTP 외 프로토콜 테스트 확장성 확보

---

### 🔄 **CI/CD 파이프라인 통합**
- GitHub Actions, GitLab CI 등과 연동
- 배포 후 자동 부하 테스트 가능
- 성능 기준 미달 시 배포 차단 등 품질 게이트 설정 가능

---

## 🟡 요약: K6를 선택해야 하는 경우

| 상황 | K6 적합 여부 |
|------|--------------|
| 단순 HTTP API 부하 테스트 | 🔶 적절 (bombardier/vegeta로도 충분) |
| 복잡한 사용자 흐름 테스트 | ✅ 매우 적합 |
| JWT 로그인 등 인증 기반 테스트 | ✅ 필수 |
| 여러 API를 시나리오로 구성 | ✅ 필수 |
| 성능 모니터링 도구 연동 | ✅ 강력한 장점 |
| 수만 RPS 이상의 부하 발생 | ✅ 분산 환경 필요 |
| CI/CD 자동화 필요 | ✅ 유리 |
| WebSocket, gRPC 등 실시간 통신 테스트 | 🔶 제한적으로 가능 |

### References

- https://github.com/grafana/k6

### 발표 후 질문

1. K6 Script 언어와 그 이유?
     * 공식적으로 Javascript, Typescript 지원
          * 왜 Javascript를 선정했나?
               * "웹 개발자 친화적이다", "러닝 커브가 낮다" 
     * 비공식적으로는 Go 지원
          * 실험적 기능https://github.com/szkiba/xk6-g0 참고
2. K6가 CLI 기반 동작인 이유?
     * CLI 명령어와 스크립트 기반 동작으로 CI/CD 통합에 최적화될 수 있는 장점이 있음
     * 숙련된 개발자를 대상으로 하는 백엔드 인프라 중점 부하테스트 도구이므로 CLI를 중점적으로 지원함
     * GUI를 생략함으로써 리소스 소비를 최소화할 수 있기 때문임
     * K6 Cloud의 경우에는 대시보드 GUI를 제공하고, Free-tier로도 제한적으로 GUI 제공되기는 함

