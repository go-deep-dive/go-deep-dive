# Go coding conventions

Go 언어는 상대적으로 단순한 언어이지만, minimalism이라는 철학을 유지하기 위해서는 나름의 코딩 컨벤션이 필요하다. Go 언어를 처음 사용하는 사람이라면 주요 문서들을 읽어보며 style convention도 함께 익히는 게 좋다. 

Google의 Go Style Decisions는 문서 내용이 길고 매우 자세한데, Go 개발자라면 지키면 좋을 Guideline이 매우 자세히 정리되어 있다.

물론 가장 좋은 방법은 이런 가이드를 직접 읽고 스스로 체화하는 것이나, 자료들이 매우 방대하기 때문에 관심 갖고 볼만한 요소들을 정리해 보았다.

글의 말미에는 style 규칙을 linter로 강제할 때 많이 사용되는 golangci-lint에 대해서도 간략히 소개한다.

# Effective Go

Go 언어 입문자들이라면 90%는 이 글을 읽을 텐데, 여기서도 몇몇 style 관련 팁을 제시한다.

[https://go.dev/doc/effective_go](https://go.dev/doc/effective_go)

## 명명규칙

### 기본 규칙

- camelCase 또는 PascalCase를 사용한다.

### Packages

- 패키지명은 lowercase이며 언더스코어(_)나 대소문자를 섞어쓰지 않는다.
- 패키지명과 디렉토리명을 통일해 쓴다
- export되는 함수나 인터페이스 struct에 패키지명을 포함하지 않는다. 중복을 피하기 위함이다.
bufio.BufReader (X) → bufio.Reader (O)
- 만약 패키지에서 export되는 생성자가 단 하나 뿐이라면 생성자는 New 로도 충분하다
[ring.New](http://ring.New)Ring (X) → ring.New (O)

### Getter

- Getter 메서드에는 “Get” Prefix를 붙이지 않는다. 
user.GetName (X) → user.Name (O)
- Setter의 경우 “Set” prefix를 붙여 구분할 수 있다.

### Interface

- 1개의 메서드만 가지는 인터페이스는 메서드명에 “er” suffix를 붙여서 이름을 만든다.
Formatter, Notifier
- string 변환 메서드는 String()으로 충분하다.
ToStr (X), ToString (X) → String (O)

## 코드 스타일

### Control flow

- if-else 대신 early-return 사용

### 기타

- blank identifier를 이용한 전역 선언을 통해 인터페이스를 체크하는 것은 해당 코드 내에 static conversion이 존재하지 않는 경우에만 한다.
`var _ json.Marshaler = (*RawMessage)(nil)`

*static conversion 예시*
```go
type Sample interface {
    A()
}

type impl struct{}

func (i impl) A() {
    return
}

func New() Sample {
    return &impl {}
}
```

- error는 error 타입으로 선언하고 사용한다.

# Google Go Style Guide

[https://google.github.io/styleguide/go/guide](https://google.github.io/styleguide/go/guide)

[https://google.github.io/styleguide/go/decisions](https://google.github.io/styleguide/go/decisions)

### Line Length

- 줄의 길이기 길다고 나누지 말 것.
    - 길다고 느껴지면 리팩토링을 할 것
    - 함수 선언이나 if/for문 등 자연스레 바뀌는 경우가 아니면 줄을 바꾸지 않음
    - 긴 문자열이라면 쪼개서 여러 개의 작은 문자열로 사용할 것

### Least mechanism

- 여러 구현이 가능한 경우 standard tool을 사용하는 옵션을 선호하라.
- 코드에 복잡성을 추가하는 것은 쉽고, 이미 복잡성을 제거하는 것은 어렵다. 따라서 불필요한 복잡성은 추가하지 말라.
    - channel, map, slice, loop 등 core language 요소를 우선 사용할 것
    - 위 요소로 해결 불가능하다면 standard library를 사용할 것

### 패키지

- 패키지명은 소문자로만 적고 underscore는 사용하지 말 것
- 패키지에 대해 정보를 제공하지 않는 패키지명은 피할 것: util, common, helper 등 피할 것
- 패키지를 import할 때 rename하는 경우, 위 규칙을 따를 것

### Receiver

- Receiver 명은 최대한 간결하게
func (tray Tray) → func (t Tray)

### 상수

- 상수명은 PascalCase로 선언
- 상수명은 그 상수의 역할에 대한 것으로 지어야 하지 상수값에 대한 설명이면 안됨

### Initialisms

- 약어나 줄임말 등은 소문자/대문자 일관되게 사용.
url (O) URL (O) Url (X)
- DDoS, gRPC, iOS 처럼 원래부터 약어나 줄임말이 대소문자가 섞인 경우, 원래 형태를 유지하되 export/unexport 여부에 따라 첫 글자만 대소문자 수정 가능

| **English Usage** | **Scope** | **Correct** | **Incorrect** |
| --- | --- | --- | --- |
| XML API | Exported | `XMLAPI` | `XmlApi`, `XMLApi`, `XmlAPI`, `XMLapi` |
| XML API | Unexported | `xmlAPI` | `xmlapi`, `xmlApi` |
| iOS | Exported | `IOS` | `Ios`, `IoS` |
| iOS | Unexported | `iOS` | `ios` |
| gRPC | Exported | `GRPC` | `Grpc` |
| gRPC | Unexported | `gRPC` | `grpc` |
| DDoS | Exported | `DDoS` | `DDOS`, `Ddos` |
| DDoS | Unexported | `ddos` | `dDoS`, `dDOS` |
| ID | Exported | `ID` | `Id` |
| ID | Unexported | `id` | `iD` |
| DB | Exported | `DB` | `Db` |
| DB | Unexported | `db` | `dB` |
| Txn | Exported | `Txn` | `TXN` |

### Getter

- get, Get을 붙이지 말 것
- HTTP GET 메서드와 관련된 것일지라도 “Get” prefix 붙이지 말 것. Getter와 혼동됨
    - 대신 Fetch prefix를 사용할 것

### Variable names

- type을 나타내는 표현을 변수명에 포함시키지 말 것
    - numUsers (X) → userCount (O)
    - userSlice(X) → users(O)
    - 단 ageString과 age가 동시에 존재하는 경우 ageString 형태로 변수명 사용하는 것도 가능
- 한 글자 변수는 의미가 불명확하므로 지양하되, 반복적으로 사용되거나 일반적으로 사용되는 케이스 등에서만 제한적으로 사용할 것
    - method의 receiver variable의 경우 한 글자로 선언 가능
    - io.Reader나 http.Request를 r로 쓰는 것은 일반적으로 사용되는 형태이므로 가능
    - for 문처럼 스코프가 작은 경우, loop identifier는 단일 글자 가능
    - i, j, x, y도 가능

### Doc comments

- top-level exported name은 반드시 doc 주석을 가져야 한다.

```go
// Options configure the group management service.
type Options struct {
    // General setup:
    Name  string
    Group *FooGroup

    // Dependencies:
    DB *sql.DB

    // Customization:
    LargeGroupThreshold int // optional; default: 10
    MinimumMembers      int // optional; default: 2
}
```

### Error

- 에러 문자열은 소문자로만, 마침표 없이 쓴다. 보통 다른 context와 함께 프린팅 되기 때문이다.

### Don’t panic

- 보통의 에러 핸들링 상황에서 panic을 쓰지 말 것
- main 패키지의 initialization 코드 등에서 프로그램을 종료해야 하는 경우 stack trace가 그다지 도움이 되지 않으므로 `log.Exit`을 쓰는 것을 고려할 것
- 불가능한 조건, 코드리뷰나 테스팅 과정에서 반드시 잡혀야 할 버그 등의 케이스에 대해서는 `log.Fatal`을 쓸 수 있다.

### Must functions

- setup helper에서는 MustXYZ와 같이 실패하면 프로그램을 종료하는 함수를 만들어 사용할 수 있다.
    - 단 프로그램의 시작 등 초기 단계에서만 사용하고, 이후 단계에서는 error를 반환해 처리하는 게 권장된다.

```go
func MustParse(version string) *Version {
    v, err := Parse(version)
    if err != nil {
        panic(fmt.Sprintf("MustParse(%q) = _, %v", version, err))
    }
    return v
}
```

### Goroutine lifetimes

- goroutine을 생성할 때 언제 종료되는지 명확히 할 것 (누수 방지)
    - 채널 블로킹, 닫힌 채널에 대한 값 전송, 데이터 레이스, goroutine 미종료로 인한 메모리 누수 등을 주의할 것
    - 간단히 go func(){}()만 사용하는 패턴은 goroutine의 라이프 사이클을 인지하기 어려우므로 주의할 것
- 좋은 goroutine 이용 팁[^1]
    - `context.Context`로 goroutine 수명을 관리하고 종료 신호 전파
    - 종료 여부를 명확히 알리는 신호 채널 사용 (채널 또는 mutex)
    - `sync.WaitGroup`을 사용하여 goroutine이 종료될 때까지 대기
    - 적절한 주석을 통해 goroutine이 종료되는 지점과 이유를 코드에 설명

### Interfaces

- interface를 implement하는 패키지는 concrete type을 반환해야 한다(보통 포인터 또는 구조체).
    - 그렇게 하는 이유는 implementation하는 측에서 필요한 추가 메서드를 interface 수정 없이 추가할 수 있기 때문임.
- 테스트 대역(test double) 구현을 export하기보다는 public API를 노출하고 그 public API를 이용해 테스트하게 하라.
- 실제로 사용되기 전까지 interface를 정의하지 마라. 실제로 어떻게 사용될지 결정되지 않은 상태에서 필요할지, 어떤게 필요할지 생각해내는 것은 너무 어렵다.
- 사용자가 다른 타입을 넣어야 하지 않는다면 interface-type의 parameter를 쓰지 말라
- 다른 패키지 사용자가 쓰지 않을 interface는 export하지 말라.

### Generics

- 불필요하게 Generic을 미리 적용해 미래의 편의를 미리 대비하지 마라. 이는 불필요하게 복잡성을 추가하게 된다. 필요할 때 generic을 적용하라.
- public API에 generic을 사용할 경우, 충분히 주석을 추가하라.

### Use %q
문자열을 formatting하는 함수를 사용하면서 내부에 따옴표로 감싸진 문자열을 넣고 싶다면 %q를 사용하라.

```go
// Good:
fmt.Printf("value %q looks like English text", someText)

// Bad:
fmt.Printf("value \"%s\" looks like English text", someText)
```

### Use any

- inferface{} 대신 any를 사용할 것

### context

- test에서 context 생성이 필요하다면 (testing.TB).Context()를 고려하라

### Test

- 동일한 코드로 많은 테스트 케이스를 테스트해야 하는 경우 table-driven test를 사용하라
    
    ```go
    // Good:
    func TestCompare(t *testing.T) {
        compareTests := []struct {
            a, b string
            want int
        }{
            {"", "", 0},
            {"a", "", 1},
            {"", "a", -1},
            {"abc", "abc", 0},
            {"ab", "abc", -1},
            {"abc", "ab", 1},
            {"x", "ab", 1},
            {"ab", "x", -1},
            {"x", "a", 1},
            {"b", "x", -1},
            // test runtime·memeq's chunked implementation
            {"abcdefgh", "abcdefgh", 0},
            {"abcdefghi", "abcdefghi", 0},
            {"abcdefghi", "abcdefghj", -1},
        }
    
        for _, test := range compareTests {
            got := Compare(test.a, test.b)
            if got != test.want {
                t.Errorf("Compare(%q, %q) = %v, want %v", test.a, test.b, got, test.want)
            }
        }
    }
    ```
    

# **Uber Go Style Guide**

### zero value mutex

```go
// Bad
mu := new(sync.Mutex)
mu.Lock()

// Good
var mu sync.Mutex
mu.Lock()
```

### Copy Slices and Maps at Boundaries

slice와 map은 내부 데이터를 가리키는 pointer를 내장하고 있기 때문에 copy를 해야하는 케이스에 주의해야 한다.

유저가 map이나 slice에 의도하지 않은 변형을 할 수 있다는 점을 늘 고려해야 한다.

```go
// Good
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// We can now modify trips[0] without affecting d1.trips.
trips[0] = ...
```

### Clean up을 위해 defer 사용

- 중간 중간에, 혹은 마지막에 clean up 하는 경우 까먹을 수 있다. defer를 이용하면 가독성도 좋고 실수도 줄어든다.

```go
// Bad: 락 해제를 까먹을 수 있음
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// Good
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count
```

### Channel Size

- 채널 사이즈는 설정이 필요하다면 1로, 아니면 설정하지 않고 unbuffered로 사용하라
    - 만약 채널 사이즈를 두고 사용해야 한다면 충분히 검토하라
- 전역변수로 에러 변수를 생성할 때는 “Err” 또는 “err”를 prefix로 붙인다.

### Errors

- 에러 매칭이 필요한 경우 (사용측에서 에러에 따라 처리해야 하는 경우) top-level error variable을 두거나 custom type을 두어 해결할 수 있다.

### 변경 가능한 전역변수

- Mutable한 전역변수 대신 Dependency Injection을 고려하라
(구조체가 값을 들고 있음)

### Public 구조체의 embedding type

- 한 번 embedding type을 쓰면, embedded된 instance의 메서드가 암묵적으로 함께 제공된다. 따라서 한번 embedding을 하면 이후에 해당 구조체에서 embedding을 제거할 수 없다. (메서드도 함께 사라져서 backward compatibility를 보전할 수 없음). 따라서 외부에 API가 노출되는 public 구조체에는 embedding type은 지양해야 한다.

```go
// BAD
type ConcreteList struct {
  *AbstractList
}

//GOOD
type ConcreteList struct {
  list *AbstractList
}

func (l *ConcreteList) Add(e Entity) {
  l.list.Add(e)
}

func (l *ConcreteList) Remove(e Entity) {
  l.list.Remove(e)
}
```

### init() 사용 지양

- init 사용을 지양하라.
- 만약 init을 사용할 경우 deterministic하게 동작해야 한다.

### os.Exit

- os.Exit과 log.Fatal은 main 함수에서만 사용하라.
    - main 함수가 아닌 곳에서 Exit, Fatal로 앱을 종료할 경우, control flow가 명확하지 않아진다.
    - 어떤 함수가 Exit, Fatal로 종료될 경우 테스트가 어려워진다.
    - Exit, Fatal 사용 시, defer로 추가된 clean up 함수들이 실행되지 않고 스킵된다.
- 다른 함수에서 사용해야할 경우에는 error를 리턴하고 메인 함수에서 처리하라.

### Exit Once

- main 함수 안에서 여러 차례 os.Exit, log.Fatal을 호출하지 말라. 최대 1번만 호출하라.
    - 만약 Exit을 해야할 여러 케이스가 있다면 이를 하나의 run 함수로 묶고 에러를 반환하게 하라. main 함수는 이 함수를 실행해 에러가 반환되면 Exit을 하게 하라.
    - 이렇게 하면 main 함수에서 분리된 함수는 테스트 가능해지며, main 함수에는 최소한의 로직만 남길 수 있다.

### Goroutine을 fire-and-forget 방식으로 사용하지 말기

- Goroutine의 lifetime은 반드시 관리돼야 한다.
    - Goroutine이 언제 종료될지 예측 가능하거나
    - 시그널을 보내 Goroutine을 종료할 수 있는 기능이 있어야 한다.

## 성능

- fmt보다 strconv로 형변환 하기
    - 메모리 할당도 적고 성능도 좋음
- 반복적인 string-to-byte 변환 지양
    
    ```go
    // BAD
    for i := 0; i < b.N; i++ {
      w.Write([]byte("Hello world"))
    }
    
    // GOOD
    data := []byte("Hello world")
    for i := 0; i < b.N; i++ {
      w.Write(data)
    }
    ```
    
- map이나 slice를 생성할 때 Capacity를 명시하기 
    - 보다 적은 런타임 메모리 할당
    - capacity가 지정된 변수가 64KB 이하라면 stack에 할당될 가능성이 높고, 이는 불필요한 GC를 줄이는 데 도움이 됨
    - capacity를 늘릴 때마다 backing array가 새로 복사되는 오버헤드가 발생

## 코드 스타일

### 패키지 작명

- 소문자로만, 언더스코어 없이 만들기
- 최대한 간결하게
- 단수형 사용 (복수형 X)
- common, util, shared, lib처럼 무의미한 이름 사용하지 않기

### 함수 순서

- export되는 함수가 private 함수보다 위에 존재해야 한다.
- 생성자 함수는 생성하는 타입 선언 바로 밑에 위치해야 한다.

### export되지 않는 전역변수

- 언더바(_)를 prefix로 붙여 의도를 명확히 드러낸다.

### nil은 valid한 slice다

- length가 zero인 slice를 만들어 return하지 말고 nil을 반환하라.
    - nil을 반환해도 nil 체크 대신 `len(s) == 0` 으로 체크해 대응 가능하다.

### Variable의 scope 줄이기

- 가능하다면 variable과 constant의 스코프를 최소한으로 유지하기 위해 노력하라

```go
if err := os.WriteFile(name, data, 0644); err != nil {
 return err
}
```

- 상수는 여러 함수에서 공통으로 사용되지 않는다면 굳이 전역 변수일 필요가 없다.

### var for zero-value struct

```go
// BAD
user := User{}

// GOOD
var user User
```

### Map 생성

- make로 만드는 것을 선호하라.

# Golangci-lint로 Style Convention을 강제하기

golangci-lint를 이용하면 팀에서 원하는 style convention을 어느 정도 강제할 수 있다.

[Introduction | golangci-lint](https://golangci-lint.run/)

예를 들어, export되는 함수들에는 반드시 주석이 있어야 한다는 룰을 적용하고 싶다면, 아래처럼 적용하면 된다.

```yaml
linters:
  enable:
    - revive
  settings:
    revive:
      rules:
        - name: exported
          severity: warning
          disabled: false
          exclude: [""]
          arguments:
            - "checkPublicInterface" # public 인터페이스 타입의 public method 주석 강제
```

그 밖에도 주요 style guide들에서 명시하는 패턴/안티패턴을 규제하는 linter들이 많으므로 쭉 읽어보고 사용할 만한 linter를 추가해 사용하면 좋다.

여러 linter들이 제공되는데, 니즈에 맞는 linter를 잘 설정한다면 꽤나 디테일한 적용도 가능하다.

예를 들면 아래처럼 regex 패턴기반으로 log.Print를 사용 금지할 수도 있다.

```yaml
linters:
  enable:
    - forbidigo
  settings:
    forbidigo:
      forbid:
        - pattern: log\.Print
          msg: "log.Print() 사용 금지"
          pkg: "^log$"
      analyze-types: true
```

golangci-lint는 로컬에서도 `golangci-lint run` 명령어로 실행할 수 있지만, github action이나 gitlab ci와도 잘 연동되며 CI에서 사용하기에도 무리가 없다.

[Install | golangci-lint](https://golangci-lint.run/welcome/install/)

아래는 유명 Go 기반 오픈소스 라이브러리들의 golangci-lint 사용 여부를 정리한 것이다.

| 라이브러리 | `.golangci.yml` 여부 |
| --- | --- |
| [kubernetes](https://github.com/kubernetes/kubernetes) | ❌ |
| [gin](https://github.com/gin-gonic/gin/blob/master/.golangci.yml) | ✔ |
| [moby](https://github.com/moby/moby/blob/master/.golangci.yml) | ✔ |
| [prometheus](https://github.com/prometheus/prometheus/blob/main/.golangci.yml) | ✔ |
| [ollama](https://github.com/ollama/ollama/blob/main/.golangci.yaml) | ✔ |
| [traefik](https://github.com/traefik/traefik) | ✔ |
| [zap](https://github.com/uber-go/zap/blob/master/.golangci.yml) | ✔ |
| [**open-match**](https://github.com/googleforgames/open-match/blob/main/.golangci.yaml) | ✔ |
| [grafana](https://github.com/grafana/grafana/blob/main/.golangci.yml) | ✔ |
| [elastic/beats(filebeat, etc)](https://github.com/elastic/beats/blob/main/.golangci.yml) | ✔ |

### 자주 사용되는 linter

golangci.yml을 사용하는 9개 라이브러리들에서 주요 linter의 사용 빈도 정리

| 린터 | 사용한 라이브러리 수 | 목적 |
| --- | --- | --- |
| misspell | 6 | 자주 스펠링 틀리는 영어 단어 체크 |
| goimports | 5 | import 추가/미사용 제거 |
| nolintlint | 5 | 잘못 작성된 nolint 선언 체크 |
| revive | 5 | golint 대체제 |
| unconvert | 5 | 불필요한 type conversion 제거 |
| depguard | 4 | 특정 패키지 import를 허용/금지 가능 |
| errorlint | 4 | error 비교시 [errors.Is](http://errors.is/) 사용 강제 등 |
| gosec | 4 | 소스코드의 보안 이슈 체크 |
| nakedret | 4 | 길이가 긴 함수의 naked return을 방지 |
| wastedassign | 4 | 불필요하게 낭비되는 변수 할당(할당 후 사용 안하는 케이스) 체크 |

# Ref

- [(공식) Effective Go](https://go.dev/doc/effective_go)
- [(Google) Go Style Guide](https://google.github.io/styleguide/go/)
- [(Uber) Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [(뱅크샐러드) Go 코딩 컨벤션](https://blog.banksalad.com/tech/go-best-practice-in-banksalad/)


[^1]: [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup)을 활용할 경우 여러 개의 `func() error`를 goroutine으로 실행하고, 작업이 '하나'라도 실패하는 경우를 체크하는 방식으로 사용할 수 있음. [코드 참고: justErrors](https://pkg.go.dev/golang.org/x/sync/errgroup#example-Group-JustErrors)
