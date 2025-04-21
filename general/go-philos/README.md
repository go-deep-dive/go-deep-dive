# Go 언어에서 배운 것들

## 메모리 정렬

### 메모리 정렬이란

Memory alignment는 CPU 레지스터가 데이터를 효율적으로 읽고 쓸 수 있도록 데이터를 메모리에서 정렬하는 방식.  

현대 CPU의 레지스터는 64bit 체제로, 메모리 접근 시 메모리 버스를 이용. 이때 64비트 단위로 전송하며, 메모리 주소를 0, 8, 16등 8의 배수일 때 최대 효율로 데이터를 읽을 수 있음.

따라서 많은 언어에서 값을 담는 변수의 메모리 주소를 8의 배수로 배치함

![asdf](unaligned.png)

예를 들면, 변수의 주소를 표현할 때 8의 배수로 표현하면 CPU 레지스터는 한 번만 값을 표현하면 됨. 그러나 8의 배수가 아닐 때는 두 번의 나눠 읽고 결합해야 함

> READ(0 - 7) + READ(8 - 15)

![fda](aligned.png)

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

![pad](padding-unaligned.png)

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

![pad](padding-aligned.png)

* 패딩이 필요하지 않아 메모리 효율화 극대


struct 메모리 정렬은 다음의 경우에 유리할 수 있음
* 임베디드 프로그래밍
* memory intensive job
  * cloud의 경우, 메모리를 절약할 수 있는 더 저렴한 머신을 사용할 수 있음


**Go 언어에서의 struct 메모리 정렬을 위한 일반적인 good practice는 크기가 작은 필드를 앞에 넣자**


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

* 공유 메모리에 직접 접근
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
    close(ch) // Close the channel when done
}

func consumer(ch chan int) {
    for item := range ch {
        fmt.Printf("Consumed: %d\n", item)
        time.Sleep(1 * time.Second) // Simulate consumption delay
    }
}

func main() {
    ch := make(chan int, 5) // Buffered channel with capacity 5

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

> Why build concurrency on the ideas of CSP?¶
Concurrency and multi-threaded programming have over time developed a reputation for difficulty. We believe this is due partly to complex designs such as pthreads and partly to overemphasis on low-level details such as mutexes, condition variables, and memory barriers. Higher-level interfaces enable much simpler code, even if there are still mutexes and such under the covers.  
One of the most successful models for providing high-level linguistic support for concurrency comes from Hoare’s Communicating Sequential Processes, or CSP. Occam and Erlang are two well known languages that stem from CSP. Go’s concurrency primitives derive from a different part of the family tree whose main contribution is the powerful notion of channels as first class objects. Experience with several earlier languages has shown that the CSP model fits well into a procedural language framework.

동시성과 멀티 스레드는 어려움으로 악명이 높았다. 이는 부분적으로 pthread의 복잡한 디자인과 뮤텍스, 조건 변수, 메모리 장벽에 대한 지나친 강조 때문이라고 생각한다. 고수준 인터페이스는 저수준에 뮤텍스 등을 두면서도 훨씬 단순한 코드를 가능케 한다.

동시성에 대한 고수준의 언어적 지원의 가장 성공적 모델은 Tonny Hoare의 Communicating Sequential Process 또는 CSP이다. CSP는 병행성 상호작용에 대한 수학적 접근에서 시작했으며 **채널을 통한 메시지 전달을 기본으로 한다.** Occam, Erlang은 이를 구현한 잘 알려진 언어이다. Go의 동시성 primitive는 CSP 계보에서 channel을 일급 객체로 둔 강력한 발상에 기반한다. 이전 언어에 대한 경험은 CSP 모델이 절차적 언어 프레임워크에 잘 맞음을 증명한다.

* 복잡한 문법을 배제하고 간결하고 명확한 코드를 작성하는 Go 철학에 부합

### 결론

* Communicate by sharing memory: "share memory"가 사용자의 책임
* Share memory by communicating: "Communication"이 사용자의 책임
  * 동시성 제공은 언어에서 책임질게!

### 참고
* https://go.dev/doc/faq#csp
* https://stackoverflow.com/questions/36391421/explain-dont-communicate-by-sharing-memory-share-memory-by-communicating
* https://go.dev/doc/codewalk/sharemem/
* https://en.wikipedia.org/wiki/Communicating_sequential_processes#cite_note-hoare1978-7