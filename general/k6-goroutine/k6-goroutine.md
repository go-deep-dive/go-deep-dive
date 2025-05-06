# K6를 통해 배우는 Goroutine과 Multi-threading의 차이

---

## 1. K6란?

K6는 **Go 언어로 개발된 부하 테스트 및 성능 테스트 도구**로, 실제 사용 환경을 시뮬레이션하는 데 최적화되어 있습니다.

* 시나리오는 JavaScript로 작성하며, CLI를 통해 실행합니다.
* 수천에서 수십만 개의 Virtual User(VU)를 고루틴으로 시뮬레이션할 수 있습니다.
* 내부적으로 JS 런타임으로 goja를 사용합니다.

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

❓**고루틴 생성 이전에 JS 런타임을 단 한 번 실행할 수는 없는가**
   - 각각의 VU 고루틴이 자체적으로 JS 런타임 인스턴스를 생성하여 코드를 실행함으로써, 독립적인 실행 환경을 가지고 상태 격리를 이룰 수 있음
   - 또한, 각각의 고루틴이 런타임 인스턴스를 지니더라도 Go의 GC를 통해 메모리 효율성을 확보할 수 있으므로 크게 문제가 되지 않음
   - 따라서, VU 고루틴이 각각 JS 런타임을 실행하여 시나리오를 파싱한 데이터를 들고 있음

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
* goja 엔진을 통해 JavaScript 파싱 및 실행
* JS의 함수(`http.get`, `sleep`, `check`)는 Go로 구현된 모듈과 연결

### 2) VU(Virtual User) 생성 및 실행

* 위치: `lib/executor/`
* 각 VU는 고루틴으로 실행되며, 루프 안에서 JS를 반복 실행
* 각 실행은 `vu.Run()`를 통해 수행됨

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
* 각 VU는 실행 결과를 `stats.Sample`로 생성 후 채널을 통해 전달

```go
vuState.Samples <- &stats.Sample{
    Metric: myMetric,
    Value:  200,
    Tags:   vuState.Tags.CloneTags(),
}
```

* Stats Engine에서는 이를 수신하여 집계 처리

```go
func (se *Engine) runAggregator() {
    for sample := range se.SampleChan {
        se.aggregator.Add(sample)
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

* VU는 고루틴으로 실행 → 경량 스레드로 수천 개도 무리 없이 실행 가능
* 통계는 채널로 전달 → 통신은 Lock-free, 처리 루틴은 단일 고루틴

---

## 5. 결론: 성능을 위해 설계된 K6

### ✅ 고루틴 기반 병렬성

* 각 VU는 고루틴으로 실행되어 **가볍고 병렬성 뛰어남**
* Go의 스케줄러가 수천 개의 고루틴을 관리할 수 있음

### ✅ 비동기 채널로 Metric 전송

* VU는 Sample을 생성한 후 즉시 채널로 전송
* 통계 집계는 별도 루틴에서 처리되어 병목 없음

### ✅ Stats Engine은 단일 루틴 처리

* Aggregator가 모든 Sample을 처리
* 락 없이 버퍼에 집계 후 주기적 Export

### 🎯 실전 병목 회피 전략

| 전략             | 설명                     |
| -------------- | ---------------------- |
| 채널 버퍼 충분 확보    | VU가 채널 전송 시 대기하지 않도록 함 |
| Sample 최소화     | 구조체 크기를 줄여 메모리 효율화     |
| Aggregator 최적화 | 버퍼에 먼저 집계 후 일괄 export  |

---

## 6. 핵심 문장 정리

> "k6는 부하 테스트 도구 스스로가 병목이 되지 않도록, 고루틴 기반 병렬성과 채널 기반 비동기 통계를 설계하여, 테스트 대상의 성능을 왜곡하지 않는 구조를 갖추고 있습니다."

---

## 7. Go의 고루틴이 쓰레드보다 유리한 이유

* **낮은 환경 의존성**: 스레드는 OS 리소스에 민감하지만 고루틴은 경량
* **간단한 동시성 구현**: Context Switch가 사용자 공간에서 이뤄짐
* **VU 무제한 생성 가능**: 고루틴 시작 스택은 약 2KB (스레드는 256KB 이상)
* **메트릭 처리 비동기화**: 논블로킹 I/O 활용으로 블로킹 없음

### GMP 모델

<image src="./gmp-model.png" width="500px">

* Go는 G들을 P가 스케줄링하고 M에 분배하여 실행
* 사용자 공간에서 스케줄링 → 매우 빠른 문맥 전환

📌 **스레드는 버스를 몰아야 하는 기사, 고루틴은 자전거**

수백 명이 자전거를 타고 가는 구조가 더 유연하고 빠릅니다.

---

## 8. 참고 코드 위치 (K6 GitHub)

| 기능         | 파일 위치                                    |
| ---------- | ---------------------------------------- |
| VU 실행 및 루프 | `lib/executor/vu_handle.go`<br> vuHandle.runLoopsIfPossible() 함수에서 VU의 반복 실행 로직을 처리합니다.        |
| Sample 생성  | `lib/netext/httpext/transport.go`<br> HTTP 요청의 RoundTrip 과정에서 메트릭 샘플을 생성합니다.                |
| Sample 전송  | `metrics/sample.go`<br> Sample 구조체와 관련된 메트릭 샘플의 정의와 처리를 담당합니다.                 |
| Stats 처리   | `internal/metrics/engine/`<br> 테스트 실행 중 수집된 메트릭을 집계하고 처리하는 로직이 포함되어 있습니다. |

---

### References

- https://github.com/grafana/k6
