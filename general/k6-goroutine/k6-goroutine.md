# K6 구조를 통해 배우는 부하 시뮬레이션, 메트릭 수집 파이프라인 설계

---

## 1. K6란?

K6는 **Go 언어로 개발된 부하 테스트 및 성능 테스트 도구**로, 실제 사용 환경을 시뮬레이션하는 데 최적화되어 있습니다.

* 시나리오는 JavaScript로 작성하며, CLI를 통해 실행합니다.
* 수천에서 수십만 개의 Virtual User(VU)를 고루틴으로 시뮬레이션할 수 있습니다.
* 내부적으로 JS 런타임으로 **goja**를 사용합니다.

### 구조 개요

```text
+------------------+
|   CLI Entry      | --> cmd/k6/main.go
+------------------+
         |
         v
+-----------------------+
|  Engine (core.Engine) |
+-----------------------+
         |
         v
+----------------------------+
| JS Runtime (goja Runner)   |
+----------------------------+
         |
         v
+----------------------+
| lib/http, lib/ws, ...|
+----------------------+
```

### 예시 시나리오 (script.js)

```js
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
k6 run script.js
```

### 주요 기능 요약

| 기능            | 설명                                |
| ------------- | --------------------------------- |
| HTTP 요청 지원    | GET, POST, PUT, DELETE 등          |
| WebSocket 테스트 | 실시간 서비스 테스트 가능                    |
| Thresholds    | SLA 조건 정의 가능                      |
| Checks        | Assertion 기능 제공                   |
| Metrics 출력    | JSON, InfluxDB, Prometheus 등으로 출력 |
| Cloud 실행      | Grafana Cloud 확장 가능               |

---

## 2. K6 내부 구조 및 폴더 설명

```text
k6/
├── core/        # 실행기, 스케줄러, 샘플러 등 핵심 로직
├── js/          # JavaScript 런타임 연동(goja)
├── lib/         # HTTP, WS 등 라이브러리 모듈
├── stats/       # 통계 수집 및 집계
├── cmd/         # CLI 명령 처리
├── ui/          # CLI 출력, 진척도 바 등
├── config/      # 설정 처리
└── modules/     # JS import 가능한 모듈들
```

진입점:

```go
func main() {
    rootCmd := commands.NewRootCommand()
    rootCmd.Execute()
}
```

---

## 3. 시뮬레이션 플로우 및 내부 동작 구조

### 1) JavaScript 스크립트 파싱

* 위치: `js/runner.go`, `modules/`
* goja 엔진을 통해 JavaScript 파싱 및 실행
* JS의 함수(`http.get`, `sleep`, `check`)는 Go로 구현된 모듈과 연결

### 2) VU(Virtual User) 생성 및 실행

* 위치: `core/engine.go`, `core/local`, `core/execution_plan.go`
* 각 VU는 고루틴으로 실행되며, 루프 안에서 JS를 반복 실행
* 각 실행은 `vu.RunLoop()`를 통해 수행됨

```go
func (vu *VU) RunLoop(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
                vu.RunOnce()
            }
        }
    }()
}
```

### 3) Metrics 수집

* 위치: `stats/engine.go`, `stats/metrics.go`
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

## 7. 참고 코드 위치 (K6 GitHub)

| 기능         | 파일 위치                                    |
| ---------- | ---------------------------------------- |
| VU 실행 및 루프 | `core/engine.go`, `lib/executor`         |
| Sample 생성  | `lib/netext/transport.go`                |
| Sample 전송  | `lib/metrics/metrics.go`                 |
| Stats 처리   | `stats/engine.go`, `stats/aggregator.go` |

---

## 8. Go의 고루틴이 쓰레드보다 유리한 이유

* **낮은 환경 의존성**: 스레드는 OS 리소스에 민감하지만 고루틴은 경량
* **간단한 동시성 구현**: Context Switch가 사용자 공간에서 이뤄짐
* **VU 무제한 생성 가능**: 고루틴 시작 스택은 약 2KB (스레드는 256KB 이상)
* **메트릭 처리 비동기화**: 논블로킹 I/O 활용으로 블로킹 없음

### GMP 모델

<image src="./gmp-model.png">

```text
G (Goroutine) → P (Processor) → M (OS Thread)
```

* Go는 G들을 P가 스케줄링하고 M에 분배하여 실행
* 사용자 공간에서 스케줄링 → 매우 빠른 문맥 전환

📌 **스레드는 버스를 몰아야 하는 기사, 고루틴은 자전거**

수백 명이 자전거를 타고 가는 구조가 더 유연하고 빠릅니다.
