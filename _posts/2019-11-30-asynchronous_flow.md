---
title: (번역) Asynchronous Flow
tags: kotlin, reactive
key : post-kotlin-flow
header:
  theme: dark
  background: 'linear-gradient(135deg, rgb(34, 139, 87), rgb(139, 34, 139))'
---

## Asynchronous Flow
[원본 링크](https://kotlinlang.org/docs/reference/coroutines/flow.html)

서스펜딩 함수들은 비동기적으로 하나의 값을 반환한다.<br>
하지만 우리는 어떻게 비동기적으로 계산된 여러 값을 반환할 수 있을까?<br>
바로 그곳이 코틀린 플로우가 필요한 부분이다.<br>

---

### Representing multiple values
여러 값들은 코틀린의 콜렉션을 이용해 표현 가능하다.<br>
예를 들면 foo 라는 3개의 숫자 리스트 반환하는 함수를 가질 수 있고,<br> 
이 결과를 foreach를 이용해 전체 값을 프린트 할 수 있다.<br>

```
fun foo(): List<Int> = listOf(1, 2, 3)
 
fun main() {
    foo().forEach { value -> println(value) } 
}

Output
1
2
3
```
#### Sequences
만약 우리가 CPU 자원을 소모하는 블록킹 코드로 몇몇의 숫자를 계산한다면<br>
(각 계산에는 100ms가 소요된다), 우리는 숫자들을 시퀀스로 표현할 수 있다.<br>

```
fun foo(): Sequence<Int> = sequence { // sequence builder
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it
        yield(i) // yield next value
    }
}

fun main() {
    foo().forEach { value -> println(value) } 
}
```
이 코드는 위에서와 동일한 숫자를 출력한다.<br>
하지만 각각을 출력하기 전에 100ms를 기다린다.<br>

#### Suspending functions
하지만 이 계산은 메인 스레드를 차단하고 코드를 실행한다.<br>
이 값들을 비동기적으로 계산하려면, foo 함수에 `suspend` 모디파이어로 비동기 코드임을 표시해야한다.<br>
그러면 블로킹 없이 결과를 리스트로 반환 할 수 있다.<br>

```
suspend fun foo(): List<Int> {
    delay(1000) // pretend we are doing something asynchronous here
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    foo().forEach { value -> println(value) } 
}
```
이 코드는 1초를 기다린 후에 숫자를 출력한다.<br>

#### Flows
List<Int> 결과 타입의 의미는 모든 값을 한번에 반환할 수 있음을 의미한다.<br>
값들의 스트림이 표현하는 것은 비동기 계산 될 수 있음을 의미한다.<br>
우리는 Flow<Int> 타입을 사용해, 동기적으로 계산된 값을 위해 Sequence<Int> 타입을 사용하는 것 처럼 사용할 수 있다.<br>

```
fun foo(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) // pretend we are doing something useful here
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    // Launch a concurrent coroutine to check if the main thread is blocked
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // Collect the flow
    foo().collect { value -> println(value) } 
}
```

이 코드는 각 숫자가 프린트 되기 전에 메인스레드의 블록킹 없이 100ms 을 기다린다.<br>
이것은 메인스레드로 부터 시작한 분리된 코루틴 코드가 실행되어 “i’m not blocked”가 매 100ms 마다 출력되는 것으로 증명된다.<br>

```
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

Flow로 작성된 코드와 이전 예제와의 차이는 다음과 같다.<br>
- Flow 타입을 만들기 위한 빌더 함수는 flow 이다.
- flow{ … } 빌더 블록 안의 코드는 suspend 될 수 있다.
- foo() 함수는 더 이상 suspend 모디파이어를 달지 않는다.
- 값들은 flow의 emit 함수로 내보낸다.
- 값들을 flow의 collect 함수로 모은다.

```
우리는 foo 의 flow { … } 몸체의 delay 를 Thread.sleep 으로 교체할 수 있다.
그러면 이 경우 메인 스레드가 차단되는 모습을 볼 수 있다.
```

### Flows are cold

Flow는 시퀀스와 유사한 콜드 스트림이다.<br>
Flow 빌더 내의 코드는 flow가 모이기 전까지 실행되지 않는다.<br>
이것은 다음 예제에서 확실히 알 수 있다.<br>

```
fun foo(): Flow<Int> = flow { 
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling foo...")
    val flow = foo()
    println("Calling collect...")
    flow.collect { value -> println(value) } 
    println("Calling collect again...")
    flow.collect { value -> println(value) } 
}
```
출력은 :
```
Calling foo...
Calling collect...
Flow started
1
2
3
Calling collect again...
Flow started
1
2
3
```

이것의 주요 이유는 foo() 함수(flow를 반환하는 것) suspend 모디파이어를 가지지 않았기 때문이다.<br>
foo() 자신은 어떤 것도 기다리지 않고 바로 반환한다.<br>
flow는 매번 모으기를 수행하는데, 이것이 우리가 collect를 호출할 때 마다 “Flow Started”가 보이는 이유이다.<br>

### Flow cancellation

Flow는 코루틴의 일반 협력 취소 방식에 따른다.<br>
하지만 flow 인프라는 추가적 취소 지점을 제공하지 않는다.<br>
이것은 취소를 위해 완전 투명하다. <br>
일반적으로 flow 콜렉션은 취소 가능한 서스펜딩 함수가 서스펜드 된 상태일 때 취소 가능하다.(delay 같은 상태)<br>
그 반대의 상황에서는 취소 할 수 없다.<br>
이어지는 예제는 어떻게 flow가 withTimeoutOrNull 블록에서 실행되는 코드가 타임아웃으로 취소 되고 멈추는지 보여주고 있다.<br>
```
fun foo(): Flow<Int> = flow { 
    for (i in 1..3) {
        delay(100)          
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // Timeout after 250ms 
        foo().collect { value -> println(value) } 
    }
    println("Done")
}
```

foo함수가 오직 두개 숫자만 flow에서 배출하여 다음과 같은 출력을 보여준다.<br>

```
Emitting 1
1
Emitting 2
2
Done
```

### Flow builders

이전 예제의 `flow { … }` 빌더는 가장 기본중 하나이다.<br>
flow를 손쉽게 정의 하기 위한 다른 빌더 또한 존재한다.<br>
<br>
flowOf 빌더는 고정된 값 세트를 방출하는 플로우를 정의한다.<br>
여러 콜렉션과 시퀀스들은 .asFlow() 확장 함수로 변환 가능하다.<br>
예시로 1에서 3까지의 숫자를 출력하는 flow를 다음과 같이 작성할 수 있다.<br>
```
// Convert an integer range to a flow
(1..3).asFlow().collect { value -> println(value) }
```

#### Intermediate flow operators

플로우는 콜렉션이나 스트림을 사용하는 것 같이 연산자에 의해 변환될 수 있다.<br>
중간 플로우는 업스트림 플로우에 적용되어 다운 스트림 플로우를 반환한다.<br>
오퍼레이터들은 플로우 자체와 마찮가지로 cold 하다.<br>
오퍼레이터의 호출 자체는 서스펜딩 함수가 아니다.<br>
빠르게 동작하여 정의된 새로운 변환된 플로우를 반환한다.<br>
<br>
기본 오퍼레이터들은 `filter`나 `map` 같은 친숙한 이름을 가지고 있다.<br>
시퀀스와의 중요한 차이점은 오퍼레이터 내의 코드블록 내에서 서스펜딩 함수를 호출할 수 있다.<br>
<br>
예를 들면, 들어오는 요청의 플로우는 서스팬딩 함수와 같은 오랜 시간<br>
동작하는 작업과도 map 연산자로 맵핑 될 수 있다.<br>
```
suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // a flow of requests
        .map { request -> performRequest(request) }
        .collect { response -> println(response) }
}
```

결과는 아래의 3개 라인으로 각 1초마다 라인이 한개씩 나타난다.<br>
```
response 1
response 2
response 3
```

#### Transform operator

flow의 변환 연산자 중에서, 가장 일반적인 것 하나는 `transform` 이다.<br>
이것은 `map`이나 `filter` 같은 간단한 변환들을 모방할 뿐 아니라 더 복잡한 변환을 구현하는데 쓰일 수 있다.<br>
`transform` 연산을 사용해, 우리는 임의의 값을 임의의 횟수만큼 배출 할 수 있다.<br>
<br>
예를 들어, `transform`을 사용해 우리는 긴 시간이 걸리는 비동기 요청 전에 문자열을 배출 하고 받을 수 있다.<br>
```
(1..3).asFlow() // a flow of requests
    .transform { request ->
        emit("Making request $request") 
        emit(performRequest(request)) 
    }
    .collect { response -> println(response) }
```

이 코드의 출력은:
```
Making request 1
response 1
Making request 2
response 2
Making request 3
response 3
```

#### Size-limiting operators

`take`와 같은 크기 제한 중간 연산자들은 제한된 시점에 도달하면 flow의 실행을 취소한다.<br>
코루틴의 취소는 항상 예외 throw에 의해 일어난다. 따라서 모든 자원 관리 함수들<br>
(try { … } finally { … } 블록 같은)은 취소가 발생할때 보통 수행된다.<br>
```
fun numbers(): Flow<Int> = flow {
    try {                          
        emit(1)
        emit(2) 
        println("This line will not execute")
        emit(3)    
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers() 
        .take(2) // take only the first two
        .collect { value -> println(value) }
} 
```

이 코드의 출력은 `numbers` 안의 `flow { … }` 바디의 실행이 두번째 숫자를 배출한 후에 멈추는 것을 명확하게 보여준다.<br>
```
1
2
Finally in numbers
```

#### Terminal flow operators

flow의 단말 연산자들은 서스펜딩 함수로, 콜렉션의 수집을 시작한다.<br>
`collect` 연산자는 가장 기본적인 연산자의 하나이지만, 그밖에도 쉽게 사용가능한 많은 다른 연산자들이 존재한다.<br>
여러 컬렉션을 변환하는 toList 나 toSet, 값을 얻을 수 있는 first <br>
그리고 하나의 flow가 값의 배출을 보장하는 single<br>
값의 플로우를 축약하는 reduce 와 fold<br>
예시:
```
val sum = (1..5).asFlow()
    .map { it * it } // squares of numbers from 1 to 5                           
    .reduce { a, b -> a + b } // sum them (terminal operator)
println(sum)
```
하나의 숫자를 출력:
```
55
```

### Flows are sequential

플로우의 개별 콜렉션은 여러 플로우에서 동작하는 특별한 연산자를 제외하고 순차적으로 수행된다.<br>
콜렉션은 단말 연산자를 호출하면 코루틴 안에서 바로 동작한다.<br>
기본적으로 새 코루틴이 시작되지 않는다. <br>
각 배출 값들은 중간 연산자들에 의해 업스트림에서 다운 스트림으로 연산되고, 그 이후 단말 연산자로 전달된다.<br>
<br>
짝수의 필터 그리고 문자열로의 맵핑을 하는 예제:<br>
```
(1..5).asFlow()
    .filter {
        println("Filter $it")
        it % 2 == 0              
    }              
    .map { 
        println("Map $it")
        "string $it"
    }.collect { 
        println("Collect $it")
    }
```
결과:
```
Filter 1
Filter 2
Map 2
Collect string 2
Filter 3
Filter 4
Map 4
Collect string 4
Filter 5
```

### Flow context
플로우의 수집은 항상 코루틴이라고 부르는 컨텍스트안에서 발생한다.<br>
예를 들면, `foo`라는 플로우가 있을 때, 구현 상세에 상관 없이<br> 
후속 코드는 코드 작성자가 지정한 특정 컨텍스트 안에서 실행된다.<br>
```
withContext(context) {
    foo.collect { value ->
        println(value) // run in the specified context 
    }
}
```
이런 플로우의 속성을 컨텍스트 보존(context preservation) 이라 불린다.<br>
<br>
기본으로, flow{ … } 빌더 내의 코드는 해당 플로우의 콜렉터가 제공하는 콘텍스트 안에서 실행된다.<br>
예를 들면, 호출 스레드를 출력 하고 3개의 숫자를 배출하는 `foo`의 구현을 생각해보자:<br>
```
fun foo(): Flow<Int> = flow {
    log("Started foo flow")
    for (i in 1..3) {
        emit(i)
    }
}  

fun main() = runBlocking<Unit> {
    foo().collect { value -> log("Collected $value") } 
} 
```
실행된 이 코드는 생산한다.
```
[main @coroutine#1] Started foo flow
[main @coroutine#1] Collected 1
[main @coroutine#1] Collected 2
[main @coroutine#1] Collected 3
```
`foo().collect` 메인 스레드에서 호출된 이후 `foo` flow의 본문 또한 메인 스레드에서 호출된다.<br>
이것은 빠른 실행 혹은 실행 컨텍스트에 상관없이 호출자가 블록되지 않는 비동기 코드를 위한 완벽한 기본이다.<br>

#### Wrong emission withContext
하지만 오랫동안 CPU를 소모하는 코드는 Dispatchers.Default 컨텍스트에서 실행될 필요가 있다.<br>
UI를 업데이트하는 코드는 Dispatchers.Main 컨텍스트에서 실행될 필요가 있다.<br>
보통, withContext는 코틀린 코루틴 코드를 사용하여 컨텍스트를 변경하는데 사용된다.<br>
하지만 `flow{ … }` 빌더내의 코드는 컨텍스트 보전 속성을 준수해야한다.<br>
다른 컨텍스트로 부터의 배출을 허용하지 않는다.<br>
<br>
이어지는 코드를 실행해보자:
```
fun foo(): Flow<Int> = flow {
    // The WRONG way to change context for CPU-consuming code in flow builder
    kotlinx.coroutines.withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100) // pretend we are computing it in CPU-consuming way
            emit(i) // emit next value
        }
    }
}

fun main() = runBlocking<Unit> {
    foo().collect { value -> println(value) } 
}
```
이 코드는 이어지는 예외를 발생시킨다.
```
Exception in thread "main" java.lang.IllegalStateException: Flow invariant is violated:
        Flow was collected in [CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@5511c7f8, BlockingEventLoop@2eac3323],
        but emission happened in [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@2dae0000, DefaultDispatcher].
        Please refer to 'flow' documentation or use 'flowOn' instead
    at ...
```
```
참고, 우리는 예제에서 예외 데모를 위해 kotlinx.coroutines.withContext 함수의 전체 이름을 사용 하였다.
withContext의 짧은 이름은 오류를 생산을 방지하는 특별한 스텁 함수를 통해 문제가 발생하지 않도록 한다.
```

#### flowOn operator
예외는 flow 배출 컨텍스트를 flowOn 함수를 사용하여 변경하도록 안내한다.<br>
flow의 컨텍스트를 변경하는 올바른 방법은 아래 예제에서 볼 수 있다.<br>
또한 모든 작업들이 어떤 스레드 이름을 가지는지 프린트 한다.<br>
```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it in CPU-consuming way
        log("Emitting $i")
        emit(i) // emit next value
    }
}.flowOn(Dispatchers.Default) // RIGHT way to change context for CPU-consuming code in flow builder

fun main() = runBlocking<Unit> {
    foo().collect { value ->
        log("Collected $value") 
    } 
} 
```
메인 스레드에서 collection을 하는 동안 어떻게 `flow { … }`가 백그라운드에서 동작하는지 알려준다:<br>
<br>
또 다른 관찰할 사항은 flowOn 연산자가 flow의 기본 순서 특성을 변경했다는 것이다.<br>
하나의 코루틴(“coroutine#1”)에서 collection이 일어나고, <br>
수집중인 코루틴과 다른 스레드에서 동시에 다른 코루틴(“coroutine#2”)에서 배출이 있어났다.<br>
flowOn 연산자는 업스트림 플로우 컨텍스트 안의 CoroutineDispatcher를 변경해야 할 때 다른 코루틴을 생성한다.<br>

### Buffering
다른 코루틴 안에서 플로우의 다른 파트를 실행하면 플로우를 수집하는 작업, <br>
특히 오랜시간 수행되는 비동기 연산이 포함된 경우 전체 수행시간 관점에서 도움이 될 수 있다.<br>
예를 들어,  `foo()` 플로우의 배출이 느린 경우, 하나의 원소를 만드는데 100ms가 소모; <br>
그리고 수집 또한 느리게, 하나의 원소를 처리하는데 300ms 소모되는 경우를 생각해보자.<br>
3개의 숫자들과 플로우를 수집하는데 얼마나 걸리는지 살펴보자.<br>
```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100) // pretend we are asynchronously waiting 100 ms
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> { 
    val time = measureTimeMillis {
        foo().collect { value -> 
            delay(300) // pretend we are processing it for 300 ms
            println(value) 
        } 
    }   
    println("Collected in $time ms")
}
```
전체 콜랙션 1200ms(3개의 숫자, 각 400ms) 과 같이 생성된다 
```
1
2
3
Collected in 1220 ms
```
우리는 플로에서 `foo()`의 배출과 수집을 순차적으로 실행하는 것과 달리<br>
동시에 실행하는데에 buffer 오퍼레이터를 사용할 수 있다:
```
val time = measureTimeMillis {
    foo()
        .buffer() // buffer emissions, don't wait
        .collect { value -> 
            delay(300) // pretend we are processing it for 300 ms
            println(value) 
        } 
}   
println("Collected in $time ms")
```
이것은 동일한 숫자들을 더 빠르게 생산한다, 효과적 생성 프로세싱 파이프라인을 가지므로,<br>
첫 숫자에 100ms만 기다리고 각 숫자를 처리하는데 오직 300ms를 소비한다.<br>
이 방법은 실행에 1000ms을 소모한다:
```
1
2
3
Collected in 1071 ms
```
```
참고 flowOn 연산은 CoroutineDispatcher을 변경해야 할 때 동일한 버퍼링 매커니즘을 사용한다. 
```

#### Conflation
플로우가 연산의 부분 결과를 표현하거나 연상 상태를 업데이트 할 때, <br>
각 값을 처리할 필요가 없지만, 대신 오직 최근 하나의 값을 처리해야 한다.<br>
수집기 처리하는데 매우 느린 경우, conflate 연산을 중간값을 생략하는데 사용할 수 있다.<br>
이전 예제를 기반으로 만들어보면:
```
val time = measureTimeMillis {
    foo()
        .conflate() // conflate emissions, don't process each one
        .collect { value -> 
            delay(300) // pretend we are processing it for 300 ms
            println(value) 
        } 
}   
println("Collected in $time ms")
```
첫번째 숫자가 몇 초간 실행되는 동안, 세번째는 이미 생산되어, 두번째는 합쳐졌다.<br>
그리고 가장 최근것(3번째)이 콜렉터에 전달된다.
```
1
3
Collected in 758 ms
```

#### Processing the latest value
콘플레이션은 에미터와 콜렉터 양쪽 모두 느릴 때 프로세싱 속도를 올리는 한 방법이다.<br>
콘플레이션은 배출된 값을 제거한다. 또 다른 방법은 새 값을 배출할 때, 느린 콜렉터를 취소하고 재실행하는 것이다.<br>
이것에는 동일한 핵심 로직 `xxx` 연산을 수행하는 `xxxLatest` 오퍼레이터 종류가 있다.<br>
하지만 새 값이 있을 경우 블록의 코드는 취소된다.<br>
이전 예제에서 `conflate`에서 `collectLatest` 바꿔보자:
```
val time = measureTimeMillis {
    foo()
        .collectLatest { value -> // cancel & restart on the latest value
            println("Collecting $value") 
            delay(300) // pretend we are processing it for 300 ms
            println("Done $value") 
        } 
}   
println("Collected in $time ms")
```
`collectLatest`의 본문에서 300ms를 소모하지만, 새 값들은 매번 100ms마다 배출된다,<br>
우리는 블록에서 실행되는 모든 값을 볼 수 있다.<br>
하지만 오직 마지막 값만 완성된다:
```
Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 741 ms
```

### Composing multiple flows
여러 플로우들을 합성하는 방법은 여러개 존재한다.<br>

#### Zip
코틀린 표준 라이브러리 내의 Sequence.zip 확장 함수처럼, 플로우는 zip 연산자를 가진다.<br>
Zip 연산자는 두개의 플로우의 값을 정확하게 결합한다:
```
val nums = (1..3).asFlow() // numbers 1..3
val strs = flowOf("one", "two", "three") // strings 
nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string
    .collect { println(it) } // collect and print
```
이 예제는 출력한다.
```
1 -> one
2 -> two
3 -> three
```

#### Combine
플로우가 변수 혹은 연산의 가장 최근 값을 나타낼 때(conflation에서 관련 섹션 참조),<br>
해당 플로우의 가장 최근 값에 기반한 연산을 수행할 필요가 있다.<br> 
그리고 어떤 업스트림 플로우에서 값을 배출할 때 재계산 해야 한다.<br>
해당 연산자 계열을 콤바인 이라고 부른다.<br>
<br>
예를 들면, 만약 이전 예제에서 숫자들을 모두 300ms로 업데이트 하고, 문자열들은 모두 400ms 후<br>
Zip 연산자를 사용하여 zipping 하면 동일한 결과를 생성해 내겠지만 출력되는 것은 모두 400ms이다:
```
이 예제에서 우리는 onEach 중간 연산자를 사용해 각 원소를 딜레이하고<br>
코드를 배출하는 예제 플로우가 더 선언적이고 짧게 만든다.
```
```
val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms
val startTime = System.currentTimeMillis() // remember the start time 
nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string with "zip"
    .collect { value -> // collect and print 
        println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
    }
```
하지만, combine 연산을 zip 대신에 여기에 사용하면:
```
val nums = (1..3).asFlow().onEach { delay(300) } // numbers 1..3 every 300 ms
val strs = flowOf("one", "two", "three").onEach { delay(400) } // strings every 400 ms          
val startTime = System.currentTimeMillis() // remember the start time 
nums.combine(strs) { a, b -> "$a -> $b" } // compose a single string with "combine"
    .collect { value -> // collect and print 
        println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
    } 
```
우리는 꽤 다른 결과를 얻는다, 출력되는 각 라인은 각 `nums` 혹은 `str` 플로우중 하나의 배출이다:
```
1 -> one at 452 ms from start
2 -> one at 651 ms from start
2 -> two at 854 ms from start
3 -> two at 952 ms from start
3 -> three at 1256 ms from start
```

### Flattening flows
플로우는 비동기적 받은 값들의 나열을 표현한다,<br>
그래서 각 값이 다른 값 시퀀스 요청을 트리거하는 상황을 마주치기 쉽다.<br> 
예를 들면, 우리는 다음과 같이 500ms 간격으로 두개의 문자열의 플로우를 반환하는 함수를 가질 수 있다:
```
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First") 
    delay(500) // wait 500 ms
    emit("$i: Second")    
}
```
이제 만약 우리가 3개의 숫자들의 플로우를 가지고 있고,<br>
다음과 같이 그들 각각이 `requestFlow`를  호출한다면..:
```
(1..3).asFlow().map { requestFlow(it) }
```
그러면 우리는 끝에 플로우의 플로우를 얻게 된다.<br>
그것은 추가 처리를 위해 하나의 플로우로 평탄화(flattened) 될 필요가 있다.<br>
콜렉션과 시퀀스는 이를 위해  flatten과 flatMap 연산자를 가진다.<br>
하지만, 동시에 플로우의 비동기 성질은 평탄화의 다른 모드를 필요로 한다.<br>
플로우에는 flattening 관련 연산자 그룹이 있다.<br>

#### flatMapConcat
결합 방법은 `flatMapConcat과 flattenConcat 연산자들에 의해 구현되어 있다.<br>
그들은 시퀀스 연산자들과 가장 직접적으로 유사하다.<br>
그들은 다음 것을 수집을 시작하기 전에 내부 플로우 완료를 기다린다.<br>
예제는 그것을 보여준다.<br>

```
val startTime = System.currentTimeMillis() // remember the start time 
(1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
    .flatMapConcat { requestFlow(it) }                                                                           
    .collect { value -> // collect and print 
        println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
    }
```
flatMapConcat의 순차 특성은 완전하게 출력으로 보여진다.
```
1: First at 121 ms from start
1: Second at 622 ms from start
2: First at 727 ms from start
2: Second at 1227 ms from start
3: First at 1328 ms from start
3: Second at 1829 ms from start
```

#### flatMapMerge
다른 평탄화(flatening) 방법은 동시에 모든 주어진 플로우를 수집한다.<br>
그리고 그 값들을 하나의 싱글 플로우로 병합하고 곧바로 배출된다.<br>
이것은 flatMapMerge 와 flattenMerge 오퍼레이터들로 구현되어 있다.<br>
그들 모두 선택적 동시적 파라미터를 채용한다. 그것은 동시성 플로우의 숫자를 제한하고,<br>
같은 시간에 모인다. (이것은 기본으로 DEFAULT_CONCURRENCY와 같다).<br>
```
val startTime = System.currentTimeMillis() // remember the start time 
(1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
    .flatMapMerge { requestFlow(it) }                                                                           
    .collect { value -> // collect and print 
        println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
    }
```
flatMapMerge 의 동시 특성은 분명하다:
```
1: First at 136 ms from start
2: First at 231 ms from start
3: First at 333 ms from start
1: Second at 639 ms from start
2: Second at 732 ms from start
3: Second at 833 ms from start
```
```
참고로 flatMapMerge은 코드 블록을 순서대로 호출한다 (이 예제에서 { requestFlow(it) }),
하지만 플로우를 동시에 수집한 결과는 먼저 map { requestFlow(it) } 을 순차적으로 수행하고
결과에서 flattenMerge를 호출하는 것과 같다.
```

#### flatMapLatest
“Processing the latest value” 섹션에서 보였던 <br>
collectLatest 연산자와 비슷한 방법에 해당하는 “최신값” 평탄화 방법이 존재한다.<br>
이전 플로우의 모음은 새로운 플로우가 배출되자 마자 취소된다.<br>
그것은 `flatMapLatest` 연산자에 의해 구현된다.
```
val startTime = System.currentTimeMillis() // remember the start time 
(1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
    .flatMapLatest { requestFlow(it) }                                                                           
    .collect { value -> // collect and print 
        println("$value at ${System.currentTimeMillis() - startTime} ms from start") 
    }
```
이 예제의 아웃풋은 flatMapLatest가 어떻게 동작하는지에 대한 좋은 예제이다.
```
1: First at 142 ms from start
2: First at 322 ms from start
3: First at 425 ms from start
3: Second at 931 ms from start
```
```
참고, flatMapLatest는 이 예제에서 block({ requestFlow(it) } 안의 모든 코드를 취소한다.
requestFlow 자신은 빠르게 호출되고 중단되지 않고 취소할 수 없어
이 부분이 다른 예제와 다르지 않아 보이게 만든다.
하지만, 만약 서스펜딩 함수를 사용한다면, 딜레이 처럼 차이를 보여줄 것이다.
```

### Flow exceptions
에미터(emitter) 혹은 코드 내의 오퍼레이터들이 예외를 던질 때,<br>
플로우 콜렉션은 예외와 함께 완료될 수 있다.<br>
이 예외들을 핸들할 수 있는 몇가지 방법이 존재한다.<br>

#### Collector try and catch
콜렉터는 코틀린 try/catch 블록을 사용하여 예외를 핸들할 수 있다.<br>
```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value ->         
            println(value)
            check(value <= 1) { "Collected $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}
```
이 코드는 성공적으로 `collect` 단말 연산자의 예외를 잡아낸다.<br>
그리고 보다시피, 그 이후 더이상 값을 배출하지 않는다.
```
Emitting 1
1
Emitting 2
2
Caught java.lang.IllegalStateException: Collected 2
```

#### Everything is caught
이전 예제는 사실 에미터 나 어떤 중간 혹은 단말 연산자들이 발생 시키는 모든 예외를 캐치한다.<br>
예를 들어, 코드를 배출된 값을 문자열로 맵핑하도록 변경해보자 하지만 코드는 예외를 생성한다.
```
fun foo(): Flow<String> = 
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }                 
        "string $value"
    }

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}      
```
이 예외는 캐치되고 콜렉션은 멈춘다:
```
Emitting 1
string 1
Emitting 2
Caught java.lang.IllegalStateException: Crashed on 2
```

### Exception transparency
하지만 에미터의 코드의 예외 핸들링 동작을 캡슐화 할 수 있을까?<br>
<br>
플로우는 예외 투명해야 한다. 그리고 try/catch 블록 내의 `flow { … }` 빌더에서<br>
값을 배출하는 것은 예외 투명성 위반이다. <br>
이것은 이전 예제에서와 같이 `try/catch`를 사용하여 콜렉터가 던지는 예외들이 잡히도록 한다.<br>
<br>
에미터는 `catch` 오퍼레이터를 사용할 수 있고 이는 예외 투명성을 보호하고 예외 핸들링의 캡슐화가 가능하게 한다. <br>
`catch` 연산자의 본문은 예외를 분석하고 잡힌 예외에 따라 다른 방법으로 반응할 수 있다.<br>
<br>
예외는 `throw`를 사용해 다시 던져질 수 있다.<br>
`catch`의 바디에서 예외를 값으로 바꾸어 배출할 수 있다.<br>
예외를 무시하거나, 기록하거나 또는 다른 처리를 할 수 있다.<br>
<br>
예를 들어, 예외를 잡았을 때 텍스트를 배출해보자.
```
foo()
    .catch { e -> emit("Caught $e") } // emit on exception
    .collect { value -> println(value) }
```
예제의 출력은 try/catch 로 코드를 감싸지 않더라도 동일한 결과를 가진다.

#### Transparent catch
예외 투명성을 고려한 `catch` 중간 연산자는 오직 업스트림 예외만을 잡는다.<br>
(`catch` 위에 있는 모든 연산자의 오류, 아래는 X).<br>
만약 `collect { … }` 안의 블록 (`catch` 아래에 위치) 이 예외를 던지면,<br>
그것은 빠져나올 것이다:
```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    foo()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}
```
“Caught …” 메시지는 ‘catch’연산자가 있음에도 불구하고 출력되지 않는다.

#### Catching declaratively
우리는 `collect`연산자의 본문을 `onEach` 안으로 이동하고 <br>
`catch`연산자의 전에 위치하게 함으로써 `catch` 연산자의 선언적 성질과 <br>
모든 예외를 핸들하고자 하는 욕구를 결합할 수 있다.<br>
플로우의 수집은 파라미터 없이 `collect()`가 호출됨으로써 트리거 된다.
```
foo()
    .onEach { value ->
        check(value <= 1) { "Collected $value" }                 
        println(value) 
    }
    .catch { e -> println("Caught $e") }
    .collect()
```
이제 우리는 “Caught …” 메시지가 프린트 되는 것을 볼수 있고,<br>
`try/catch`블록의 명시 없이 모든 예외를 잡을 수 있게 되었다:

### Flow completion
플로우 콜렉션이 완료되었을 때(정상적으로 또는 예외적으로) 하나의 액션 실행이 필요하다.<br>
이미 알다시피, 두가지 방법으로 완료할 수 있다: 암시적 또는 선언적<br>

#### Imperative finally block
`try/catch`에 추가로, 콜렉터는 `collect 완료 액션을 수행하기 위해 `finally`블록을 사용할 수 있다.
```
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}
```
이 코드는 세개의 숫자를 생산하는 `foo()` 플로우에 이어 “Done” 문자열을 출력한다.
```
1
2
3
Done
```
#### Declarative handling
선언적 접근법을 위해, 플로우는 `onCompletion` 중간 연산자를 가지고 있다.<br>
이 연산자는 플로우가 수집을 마쳤을때 실행된다.<br>
<br>
이전 예제는 onCompletion 연산자로 다시 작성될수 있고, 동일한 출력을 낸다.
```
foo()
    .onCompletion { println("Done") }
    .collect { value -> println(value) }
```
`onCompletion`의 핵심 장점은 람다의 `Throwable` 파라미터가 널 가능하여<br>
플로우의 수집이 정상적으로 완료되었는지, 예외에 의해 종료했는지 결정하는데 사용할 수 있다.<br>
이어지는 예제에서 `foo()` 플로우는 숫자 1을 배출한 후에 예외를 던진다.
```
fun foo(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}
```
예상하듯이 출력은 다음과 같다.
```
1
Flow completed exceptionally
Caught exception
```
`onCompletion` 연산자는 `catch`와 다르게 예외를 다루지 않는다.<br>
위의 코드에서 보다시피 예외는 아직 다운 스트림으로 흐른다.<br>
그것은 `onCompletion` 연산자 너머로 전달될 수 있고,<br>
`catch` 연산자에서 핸들 될 수 있다.<br>

#### Upstream exceptions only
`catch` 연산자와 유사하게, `onCompletion`은  업스트림으로 부터 오는 예외를 본다.<br>
그리고 다운 스트림의 예외는 보지 못한다. 예를 들어, 아래 코드를 실행해보자.
```
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}
```
우리는 종료의 원인이 null 이고, 수집은 예외와 함께 실패했음을 볼 수 있다.
```
1
Flow completed with null
Exception in thread "main" java.lang.IllegalStateException: Collected 2
```

### Imperative versus declarative
이제 우리는 플로우를 수집하는 법, 그리고 완료와 예외를 암시적<br>
그리고 선언적 방법으로 핸들하는 법을 알았다.<br>
여기서 자연스러운 질문은, 어떤 접근 방식이 선호되고 그 이유는 무엇인가?<br>
라이브러리로서 우리는 어떤 접근법을 지지하지 않는다. 그리고 두 옵션 모두 유효하며<br>
자신의 선호와 코드 스타일에 따라 선택해야 한다고 믿는다.

### Launching flow
플로우를 사용하여 어떤 소스로 부터 오는 비동기 이벤트 표현하는 것은 쉽다.<br>
이 경우, 우리는 한 피스의 코드를 등록과 들어온 이벤트에 대한 리액션 <br>
그리고 거슬러 올라가 작업을 지속할 수 있게 하는 `addEventListener` 함수와 유사한 것이 필요하다.<br>
`onEach` 오퍼레이터 이 역할을 할 수 있다. 하지만, `onEach`는 중간 연산자이다.<br>
우리는 또한 플로우를 모을 단말 연산자가 필요하다.<br>
하지만, `onEach`를 단순히 부른다고 하더라도 아무런 효과가 없다.<br>
<br>
만약 우리가 onEach 후에 collect 단말 연산자를 사용하면, 그 코드는 플로우가 모두 모일때까지 기다릴 것이다:
```
// Imitate a flow of events
fun events(): Flow<Int> = (1..3).asFlow().onEach { delay(100) }

fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .collect() // <--- Collecting the flow waits
    println("Done")
}
```
보다시피, 다음과 같이 출력한다:
```
Event: 1
Event: 2
Event: 3
Done
```
`launchIn` 터미널 오퍼레이터는 이럴때 편리하다.<br>
`collect`를 `launchIn`으로 교체하여 별개의 코루틴에서 플로우의 수집을 시작할 수 있다.<br>
그러므로 후속 코드의 실행이 즉각 계속된다:
```
fun main() = runBlocking<Unit> {
    events()
        .onEach { event -> println("Event: $event") }
        .launchIn(this) // <--- Launching the flow in a separate coroutine
    println("Done")
}
```
이것은 출력한다:
```
Done
Event: 1
Event: 2
Event: 3
```
`launchIn`에는 필요 파라미터로 수집 플로우를 시작할 코루틴 스코프를 명시해야 한다.<br>
위의 예제에서 이 스코프는 `runBlocking` 코루틴 빌더로 부터 온다, 플로우가 실행되는 동안,<br>
이 `runBlocking` 스코프는 차일드 코루틴의 완료를 기다린다. 그리고 메인 함수의 반환과 종료를 막는다.<br>
<br>
실제 어플레케이션에서 스코프는 제한된 라이프 타임을 가진 하나의 엔티티로 부터 온다.<br>
엔티티의 라이프 타임이 완료 되자마자, 관련된 스코프는 취소되고, 플로우의 콜렉션도 취소된다.<br>
이처럼 `onEach { … }.launchIn(scope)`의 쌍은 `addEventListener`같이 동작한다.<br>
하지만, 취소를 위해 상응하는 `removeEventListener` 함수를 필요로 하지 않는다.<br>
그리고 구조적 동시성은 이런 목적을 달성한다.<br>
<br>
참고로 `launchIn 또한 `Job`을 반환한다, 그것은 상응하는 플로우 콜렉션 코루틴을 <br>
전체 스코프의 취소나 조인 없이 `cancel`하는데 사용할 수 있다.<br>

### Flow and Reactive Streams
리액티브 스트림 또는 RxJava 나 Reactor 프로젝트 같은 리액티브 프레임워크에 친숙한 사람이라면,<br> 
플로우의 디자인은 친숙해 보일 것이다.<br>
<br>
실제로, 플로우의 디자인은 리엑티브 스트림 그리고 그 여러 구현체에서 영감을 받았다.<br>
하지만 플로우의 주요 목표는 가능한 간단한 디자인을 가지는 것과, <br>
코틀린과 서스펜션 친화적이며 동시성 구조를 존중하는 것이다.<br>
<br>
이 목표를 달성하는 것은 반응성 개척자와 그들의 거대한 작업 없이는 불가능 하다.<br>
리액티브 스트림과 코틀린 플로우 아티클에서 완전한 스토리를 읽을 수 있다.<br>
<br>
완전히 같지 않지만, 개념적으로, 플로우는 리액티브 스트림 이며 <br>
리액티브(TCK 룰 스팩) 퍼블리셔로 변환하거나 그 반대가 가능하다. <br>
이런 컨버터들은 `kotlinx.coroutines`에서 제공한다.<br>
정확히 일치하는 것은 리액티브 모듈에서 찾을 수 있다. <br>
(리액티브 스트림은 kotlinx-coroutines-reactive, <br>
리액터 프로젝트는 kotlinx-coroutines-reactor, <br>
RxJava2에는 kotlinx-coroutines-rx2)<br>
<br>
통합 모듈은 플로우 로의.. 플로우로 부터의 변환, 리엑터의 `Context` <br>
그리고 서스펜션 친화적 동작과 여러 리액티브 엔티티 등을 포함한다.<br>
