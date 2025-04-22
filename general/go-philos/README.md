# Go 언어에서 배운 것들

## 언어 디자인의 원칙

### 일반 원칙

Go를 디자인할 때, 적어도 구글에서는 Java와 C++가 서버를 작성할 때 가장 많이 사용되는 언어였다. 우리는 이 언어는 bookkeeping(부기)와 반복이 너무 많이 필요하다고 느꼈다. 이에 대해 일부 프로그래머는 효율성과 타입 안정성을 포기하며 동적이고 유동적인 파이썬 같은 언어로 옮겨가는 식으로 대응했다. **우리는 한 언어가 효율성, 안정성, 유동성을 모두 겸비할 수 있으면 했다.**

Go는 이 모두를 위해 타이핑의 양을 줄이려고 시도한다. Go의 디자인 내내 복잡함과 불필요한 코드를 덜어내려 애썼다. forward declaration을 없애려 했고 헤더 파일도 없다. 모든 것은 한 번에 선언된다. 초기화는 표현이 풍부하고 자동적이며 사용하기 쉽다. 문법은 깔끔하고 언어의 키워드가 가볍다. `(foo.Foo* myFoo = new(foo.Foo))`같은 반복은 `:=` 의 사용으로 단순하게 하려 했다. 가장 급진적으로는, 타입 위계가 없다. 그 본인일 뿐이며, 관계를 표현하지 않아도 된다. 이 단순함이 Go가 표현이 풍부하면서도 이해가 쉽고 생산성을 높일 수 있게 한다.

다른 중요한 원칙은 **개념이 직교(orthogonal)하도록 했다. 메소드는 어떤 타입을 위해서든 구현 가능하고, 구조체는 데이터를 표현하고 인터페이스는 추상성을 표현하는 등 각각은 구분되며 영향 받거나 종속되지 않는다. Orthogonality가 이 개념들이 결합할 때 무슨 일이 일어날지 예측 가능하게 한다.**

* forward declarations : 함수가 쓸 다른 함수의 선언이 반드시 사용 함수 위에 있어야 하는 것 


### Go 언어에서 exception이 없는 이유

Exception을 try - catch - finally와 같은 제어 구조체(control structure)에 결합하는 건 코드를 난해하게 만든다고 믿는다. 이는 또한 프로그래머가 파일이 안 열리는 경우가 같이 흔히 발생할 수 있는 에러를 예외적이라고 규정하게 한다고 생각한다.

Go는 다른 접근법을 취한다. 일반 에러 핸들링은 Go의 multi-value을 하고, 이는 리턴값을 overload하지 않으며 에러를 report할 수 있게 한다. Go의 다른 기능과 결합된 일반적인 형태(canonical)의 error 타입이 있어 에러 핸들링을 즐겁게 하면서도 다른 언어와 다르게 한다.

Go는 동시에 프로그램에 크리티컬한 예외 상황을 signal하고 recover하는 built-in 기능도 있다. recovery 메카니즘은 함수의 상태가 에러로 무너지는 일부분으로 실행된다. 이는 재해에 가까운 예외를 다루기 충분하면서도 별도의 control structure가 필요없으며, 잘 쓰이면 깔끔한 에러 핸들링 코드를 작성하는 데 도움이 된다. 

#### Go의 에러에 대해

**go는 내장된 interface**

```go
type error interface {
    Error() string
}
```
error는 인터페이스 타입이며, `Error()`를 구현한 모든 것은 error가 된다. error 타입은 built-in이며, `universe block`에 `predeclared`되어 있다.


#### 반복적인 에러 핸들링을 단순화하기~

go 언어에서 에러 핸들링은 중요하다. 언어 디자인과 컨벤션이 명시적으로 에러를 체크하도록 장려한다. 몇몇 경우 이 패턴이 Go 코드를 verbose하게 하는데, 다행히도 이를 최소화할 몇몇 기법이 있다.

```go
func init() {
    http.HandleFunc("/view", viewRecord)
}

func viewRecord(w http.ResponseWriter, r *http.Request) {
    // appengine: DB에서 record를 query해 template로 포매팅
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    if err := viewTemplate.Execute(w, record); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

이런 식으로 error catch가 늘어날 수 있다. 이 정도는 양반으로 계속계속 늘어나며 copy paste의 지옥을 볼 수도 있다.

```go
type appError struct {
    Error   error
    Message string
    Code    int
}
type appHandler func(http.ResponseWriter, *http.Request) *appError

func viewRecord(w http.ResponseWriter, r *http.Request) *appError {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        return &appError{err, "Record not found", 404}
    }
    if err := viewTemplate.Execute(w, record); err != nil {
        return &appError{err, "Can't display record", 500}
    }
    return nil
}

func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if e := fn(w, r); e != nil { // e is *appError, not os.Error.
        c := appengine.NewContext(r)
        c.Errorf("%v", e.Error)
        http.Error(w, e.Message, e.Code)
    }
}


func init() {
    http.Handle("/view", appHandler(viewRecord))
}
```

### 왜 타입 상속이 없을까요?

객체 지향 프로그래밍은 적어도 패러다임 중에 가장 잘 알려져있지만 타입간 관계에 대한 너무 많은 토론이 필요하며 관계가 자동적으로 생겨나기도 한다. Go는 다른 접근을 취했다.

두 타입이 관련 있음을 미리 선언하도록 요구하는 대신, Go에서 타입은 인터페이스의 메소드를 만족하기만 하면 된다. 불필요한 부기를 줄일 뿐 아니라, 추가적인 장점이 더 있다. 타입은 여러 인터페이스를 동시에 만족할 수 있어 기존의 복잡한 다중 상속의 복잡함을 피할 수 있다. **인터페이스는 매우 가벼울 수 있으며, 메소드가 없거나 하나뿐인 인터페이스가 개념을 표현하는 데 유용하다. 인터페이스는 구현 이후 새 아이디어가 도출됐거나 테스트를 위해 추가될 수도 있다.** 타입과 인터페이스간 명시적인 관계가 없기 때문에 관리하거나 논쟁할 위계가 없다.


### Go가 암묵적인 숫자 형변환을 지원하지 않는 이유

C의 숫자간 자동 형변환의 이점보다 이로 인해 발생하는 혼란이 더 크다. unassigned인지 어떻게 알지? 오버플로가 발생하나? 결과값이 아키텍쳐에 독립적인가, 이식 가능한가(portable)... 또한 이는 컴파일러 개발을 어렵게 하며, 아키텍쳐마다 일관되지 않는다. 

코드의 이식성을 보장하기 위해 몇몇 상황에서 명확한 형변환일지라도 반드시 일관되게 형 변환을 강제했다. 물론 Go의 상수가 이 문제를 좀 개선하기는 한다.

### maps, slices, channel은 reference인데 array는 왜 Values죠?

긴 이야기가 있지... 개발 당시 maps, channel는 문법적으로 pointer로 계획되었고 비 포인터 객체를 선언하고 쓸 수 없었어. array도 많이 고민했는데. 결국 우리는 pointer, values의 엄격한 분리가 언어를 사용하기 어렵게 만든다고 생각했어. 이 타입을 연결된 공유된 자료구조에 대한 referecene(descriptor)로서 동작하게 해서 이 이슈를 풀었어. 물론 이 변화가 언어의 슬픈 복잡함을 추가했지만, 사용성과 생산성은 크게 향상되었다고 생각해. 

* pointer: 변수의 메모리 주소
* reference: 내부 맵, 슬라이드 등의 데이터에 대한 포인터 등을 포함하는 descriptor (메타데이터). 포인터처럼 `&`, `*`를 쓰지 않아도 된다.

#### golang의 runtime 메모리 관리

POSIX의 프로세스 메모리 자료구조는 일반적으로 다음과 같다:
![classical-process-memory-structure](classical.webp)

* 상수, 코드: 코드 및 상수, static 값 저장 공간. 보통 작고 컴파일 시 결정
* heap: 런타임 시에 크기가 유동적으로 조절되고, 포인터, 배열 및 그외 자료구조 저장 공간. 메모리 소모량이 큰 값은 heap에서 관리
* stack: stack frame으로 구성되며 함수가 호출될 때마다 stack에 push됨. 함수가 종료할 때 stack pop
  - **스택이 하나임은 오직 하나의 실행 상태만을 가정하고 있음. 이는 멀티스레딩에는 적합하지 않음 -> 각 스레드에 로컬 변수가 많을 수 있다. 다시 말해 기술적으로 스레드의 개수가 제한될 수 있다.**

이때 기존 프로세스 모델은 stack이 8MB 정도로 작음. 이는 멀티스레드나 재귀 등을 통해 함수를 매우 많이 실행해야 할 때 stack overlfow가 발생할 수 있음. 즉, stack size도 동적이어야 한다. 

golang의 언어적 메모리 모델

![golang-process-model](go-style-memory.webp)

* **goroutine 하나마다 하나의 스택**
* heap에서 메모리를 대여하여 stack의 크기를 동적으로 변경할 수 있음. 이를 통해 수만개의 goroutine을 실행 가능케한다.
* 물론 이는 POSIX 프로세스 메모리 구조 위에서 동작하는 언어 차원의 메모리 동작 구조(like JVM)


### 참고
* https://go.dev/doc/faq#principles
* https://go.dev/blog/error-handling-and-go
* https://medium.com/@edwardpie/understanding-the-memory-model-of-golang-part-2-972fe74372ba


---

## 메모리 정렬

### 메모리 정렬이란

Memory alignment는 CPU 레지스터가 데이터를 효율적으로 읽고 쓸 수 있도록 데이터를 메모리에서 정렬하는 방식.  
현대 CPU의 레지스터는 64bit 체제로, 메모리 접근 시 메모리 버스를 이용. 이때 64비트 단위로 전송하며, 메모리 주소를 0, 8, 16등 8의 배수일 때 최대 효율로 데이터를 읽을 수 있음.

따라서 많은 언어에서 값을 담는 변수의 메모리 주소를 8의 배수로 배치.

![unaligned-memory](unaligned.png)

예를 들면, 변수의 주소를 표현할 때 8의 배수로 표현하면 CPU 레지스터는 한 번만 주소를 읽으면 됨. 그러나 8의 배수가 아닐 때는 두 번의 걸쳐 나눠 읽고 결합해야 함

> READ(0 - 7) + READ(8 - 15)

![aligned-memory](aligned.png)

반면 메모리 주소가 8바이트 단위로 최적화되어 있을 때 메모리 접근을 1번으로 최소화할 수 있음

> READ(0 - 7)

### Go 언어에서의 메모리 정렬

Go struct에서는 필드 순서에 따라 메모리 효율성이 달라진다.

```go
package main

import (
    "fmt"
    "unsafe"
)

type Unaligned struct {
    A byte
    B int64
    C [7]byte
}

func main() {
    a := Unaligned{1, 2, 3}

    fmt.Printf("Size of a is %d", unsafe.Sizeof(a))
}

Size of a is 24
```

![unaligned-struct](padding-unaligned.png)

* 필드 순서에 따라 메모리 패딩이 붙어 공간 낭비


```go
type Aligned struct {
    A byte
    C [7]byte
    B int64
}

func main() {
    a := Aligned{1, 2, 3}
    fmt.Printf("Size of b is %d", unsafe.Sizeof(b))
}

Size of a is 16
```

![aligned-struct](padding-aligned.png)

* 패딩이 필요하지 않아 메모리 효율화 극대


struct 메모리 정렬은 다음의 경우에 유리할 수 있음
* 임베디드 프로그래밍
* memory intensive job
  * cloud의 경우, 메모리를 절약할 수 있는 더 저렴한 머신을 사용할 수 있음


**Go 언어에서의 struct 메모리 정렬을 위한 일반적인 good practice는 크기가 작은 필드를 앞에 두자**


## [Share Memory By Communicating](https://go.dev/doc/codewalk/sharemem/)

### 일반적인 언어의 동시성 처리 (Communicate by sharing memory)

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Queue struct {
    data []int
    lock sync.Mutex
}

func (q *Queue) Enqueue(item int) {
    q.lock.Lock()
    defer q.lock.Unlock()
    q.data = append(q.data, item)
    fmt.Printf("Produced: %d\n", item)
}

func (q *Queue) Dequeue() int {
    q.lock.Lock()
    defer q.lock.Unlock()
    if len(q.data) == 0 {
        return -1 // Queue is empty
    }
    item := q.data[0]
    q.data = q.data[1:]
    fmt.Printf("Consumed: %d\n", item)
    return item
}

func main() {
    queue := &Queue{}

    // Producer
    go func() {
        for i := 0; i < 10; i++ {
            queue.Enqueue(i)
            time.Sleep(500 * time.Millisecond) // Simulate production delay
        }
    }()

    // Consumer
    go func() {
        for {
            item := queue.Dequeue()
            if item != -1 {
                time.Sleep(1 * time.Second) // Simulate consumption delay
            }
        }
    }()
}
```

쓰레드(goroutine)간 직접적인 데이터(data) 공유를 통해 소통한다.  
이 방식은 다음과 같은 특징이 있다.

* **공유 메모리에 직접 접근**
* 동기화 로직을 명시적으로 관리해야 하며 Error-prone
* race condition 위험


### Go의 선호되는 동시성 처리 (Share memory by communcating)


```go
package main

import (
    "fmt"
    "time"
)

func producer(ch chan int) {
    for i := 0; i < 10; i++ {
        ch <- i
        fmt.Printf("Produced: %d\n", i)
        time.Sleep(500 * time.Millisecond) // Simulate production delay
    }
    close(ch)
}

func consumer(ch chan int) {
    for item := range ch {
        fmt.Printf("Consumed: %d\n", item)
        time.Sleep(1 * time.Second) // Simulate consumption delay
    }
}

func main() {
    ch := make(chan int, 5)

    // Start producer and consumer
    go producer(ch)
    go consumer(ch)

    // Prevent the main function from exiting immediately
    select {}
}
```

* 직접적인 데이터가 아니라 채널을 통해 데이터 전달
* 채널이 동기화를 처리
* race condition 위험 최소화


### docs 

동시성과 멀티 스레드는 어려움으로 악명이 높았다. 이는 부분적으로 pthread의 복잡한 디자인과 뮤텍스, 조건 변수, 메모리 장벽에 대한 지나친 강조 때문이라고 생각한다. 고수준 인터페이스는 저수준에 뮤텍스 등을 두면서도 훨씬 단순한 코드를 가능케 한다.

동시성에 대한 고수준의 언어적 지원의 가장 성공적 모델은 Tonny Hoare의 Communicating Sequential Process 또는 CSP이다. CSP는 병행성 상호작용에 대한 수학적 접근에서 시작했으며 **채널을 통한 메시지 전달을 기본으로 한다.** Occam, Erlang은 이를 구현한 잘 알려진 언어이다. Go의 동시성 primitive는 CSP 계보에서 channel을 일급 객체로 둔 강력한 발상에 기반한다. 이전 언어에 대한 경험은 CSP 모델이 절차적 언어 프레임워크에 잘 맞음을 증명한다.

* 복잡한 문법을 배제하고 간결하고 명확한 코드를 작성하는 Go 철학에 부합

### 결론

* Communicate by sharing memory: `share memory`가 사용자의 책임
* Share memory by communicating: `Communication`이 사용자의 책임
  * 동시성 제공은 언어에서 책임질게!

### 참고
* https://go.dev/doc/faq#csp
* https://stackoverflow.com/questions/36391421/explain-dont-communicate-by-sharing-memory-share-memory-by-communicating
* https://go.dev/doc/codewalk/sharemem/
* https://en.wikipedia.org/wiki/Communicating_sequential_processes#cite_note-hoare1978-7