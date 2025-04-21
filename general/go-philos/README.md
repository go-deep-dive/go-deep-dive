# Go 언어에서 배운 것들

## 메모리 정렬

### 메모리 정렬이란

Memory alignment는 CPU 레지스터가 데이터를 효율적으로 읽고 쓸 수 있도록 데이터를 메모리에서 정렬하는 방식.  

현대 CPU의 레지스터는 64bit 체제로, 메모리 접근 시 메모리 버스를 이용. 이때 64비트 단위로 전송하며, 메모리 주소를 0, 8, 16등 8의 배수일 때 최대 효율로 데이터를 읽을 수 있음.

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

