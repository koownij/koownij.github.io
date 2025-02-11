---
title:  "[Kotlin] Coroutine context and dispatchers"
excerpt: "Coroutine context and dispatchers"

categories:
  - Kotlin
tags:
  - [코루틴]

permalink: /kotlin/Coroutine-context-and-dispatchers/

toc: true
toc_sticky: true

date: 2022-08-19
last_modified_at: 2023-02-27
---
> 코루틴 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 번역이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

<br>

코루틴은 코틀린 표준 라이브러리에 정의된 CoroutineContext type으로 표시되는 Context에서 실행됩니다.   

coroutine context는 다양한 요소들의 집합입니다. 주요 요소는 코루틴의 **Job**과 dispatcher입니다.

## Dispatchers and threads
coroutine context는 해당 코루틴이 어떤 thread(s)를 사용할 지 결정하는 *conroutine dispatcher*를 포함합니다. coroutine dispatcher는 코루틴 실행을 특정한 thread로 제한하거나 thread pool로 dispatch하거나 제한을 하지 않을 수 있습니다.   
launch나 async같은 모든 코루틴 빌더는 CoroutineContext 매개변수를 허용합니다. CoroutineContext는 새로운 코루틴과 다른 context 요소를 위한 dispatcher를 명시적으로 지정합니다.   
```kotlin
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
    println("Default               : I'm working in thread ${Thread.currentThread().name}")
}
launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
    println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
}
```
    Unconfined            : I'm working in thread main @coroutine#3
    Default               : I'm working in thread DefaultDispatcher-worker-1 @coroutine#4
    main runBlocking      : I'm working in thread main @coroutine#2
    newSingleThreadContext: I'm working in thread MyOwnThread @coroutine#5

`launch {...}`은 parameters가 없기 때문에 상위 CoroutineScope의 context를 상속 받습니다. 위의 경우 main 스레드에서 동작하고 있는 main runBlocking의 context를 상속 받습니다.   

<br>

`Dispatchers.Unconfined`는 특별한 dispatcher입니다. `main` 스레드에서 실행되지만 사슬은 다른 메커니즘을 가지고 있습니다. 해당 내용은 나중에 설명하겠습니다.

<br>

default dispatcher는 명시적으로 지정된 dispatcher가 없을 때 사용됩니다. 이는 `Dispatchers.Default`이고 스레드의 공유된 백그라운드 풀을 사용합니다.

<br>

`newSingleThreadContext`는 코루틴이 실행되기 위한 스레드를 만듭니다. 전용 스레드는 아주 비싼 리소스입니다. 실제 application에서 released되고 더이상 쓸모없게 된다면 close 함수를 사용하거나 상위 레벨의 변수에 저장하고 재사용해야합니다.

<br>

## Unconfined vs confined dispatcher
`Dispatchers.Unconfined` 코루틴 dispatcher는 첫 번재 suspension 지점까지 코루틴을 caller 스레드 안에서 시작합니다. suspension 이후 suspending 함수에 의해 결정된 스레드의 코루틴을 재개합니다. unconfined dispatcher는 CPU 시간을 소비하거나 특정 스레드에 제한된 공유 데이터(UI 등)를 업데이트하지 않는 코루틴에 적합합니다.

<br>

반면 기본적으로 dispatcher는 outer CoroutineScope에서 상속 받습니다. runBlocking 코루틴을 위한 기본 dipatcher는 스레드 호출자에 제한됩니다.따라서 상속은 예측 가능한 FIFO 스케줄링으로 이 스레드에 실행을 제한하는 효과가 있습니다.
```kotlin
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
    delay(500)
    println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
}
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
    delay(1000)
    println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
}
```


    Unconfined      : I'm working in thread main
    main runBlocking: I'm working in thread main
    Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
    main runBlocking: After delay in thread main

그래서 `runBlocking {...}` 에서 상속받은 코루틴은 `main` 스레드에서 계속되지만, unconfined 스레드는 delay 함수가 사용하는 default executor 스레드에서 재개됩니다.   

> unconfined dispatcher는 특정한 corner case에서 도움이 되는 advanced machanism입니다. 코루틴 실행 이후에 dispatching하는 것은 필요하지 않고 예상치 못한 side-effects를 발생합니다. 코루틴의 몇몇 작업은 즉시 실행되야하기 때문입니다. unconfined dispatcher는 일반적인 코드에서 사용되선 안됩니다.

<br>

## Debugging coroutines and threads
코루틴은 하나의 스레드에서 suspend되었다가 다른 스레드에서 재개될 수 있습니다. 특별한 tooling을 가지고 있지 않다면, 심지어 single-threaded dispatcher에서도 코루틴이 무엇을 하고 있는지, 어디에 있는지 파악하기 어렵습니다.   

### Debugging with IDEA
코틀린 플러그인의 Coroutine Debugger는 코루틴 디버깅을 쉽게 만듭니다.   

**Debug** tool window는 **Coroutines** 탭을 포함하고 있습니다. 해당 탭에서 실행 중이고 suspend된 코루틴의 정보를 찾을 수 있습니다. 코루틴들은 실행 중인 dispatcher에 그룹화되어 있습니다.   
![](https://kotlinlang.org/docs/images/coroutine-idea-debugging-1.png)   

coroutine debugger로 다음과 같은 작업을 할 수 있습니다.   

* 각 코루틴의 상태 체크
* running 그리고 suspend된 코루틴의 local, captured 값 확인
* coroutine creation stack 전부, 코루틴 안의 call stack 확인. stack은 변수의 모든 frames을 포함하고 있음. 이것들은 standard debugging 중에 유실될 수 있음.
* 각 코루틴의 상태와 stack에 대한 모든  보고서를 얻을 수 있다. **Coroutines** 탭을 우클릭한 후 **Get Coroutines Dump**를 클릭하면 된다.   

코루틴 디버깅을 시작하기 위해, breakpoints를 설정하고 application을 debug 모드로 실행해라.   

### Debugging using logging   
Coroutine Debugger 없이 스레드가 있는 application을 디버깅하는 다른 방법은 각각의 log 문에 thread name을 출력하는 것이다. 해당 기능은 logging이 되는 frameworks에서 모두 지원한다. coroutines를 사용할 때, 스레드 이름 만으로는 많은 정보를 제공하지 않습니다. 그래서 `kotlinx.coroutines`는 debugging facilties를 제공합니다.   

JVM 옵션에서 `-DKotlinx.coroutines.debug`와 함께 코드를 실행해보세요.
```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")    
}
```   

`runBlocking`안의 메인 코루틴(#1)과 `a`, `b` deferred 값을 계산하는 2, 3번째 코루틴(#2, #3)이 있습니다. 이것들은 전부 `runBlocking`안의 context안에서 실행되고 main thread에 한정되어 있습니다. 출력 결과는 다음과 같습니다.   

    [main @coroutine#2] I'm computing a piece of the answer
    [main @coroutine#3] I'm computing another piece of the answer
    [main @coroutine#1] The answer is 42   

`log` 함수는 대괄호와 함께 thread 이름을 출력합니다. 현재 실행 중인 코루틴이 추가된 `main` 스레드 식별자를 확인할 수 있습니다. 이 식별자는 debugging mode가 켜져있을 때 모든 생성된 코루틴에 연속해서 할당됩니다.

## Jumping between threads
`-Dkotlinx.coroutines.debug` JVM 옵션과 함께 다음 코드를 실행하라   
```kotlin
newSingleThreadContext("Ctx1").use { ctx1 ->
    newSingleThreadContext("Ctx2").use { ctx2 ->
        runBlocking(ctx1) {
            log("Started in ctx1")
            withContext(ctx2) {
                log("Working in ctx2")
            }
            log("Back to ctx1")
        }
    }
}
```   
새로운 테크닉들을 보여주고 있습니다. 하나는 `runBlocking`이 명백하게 구체화된 context에서 사용하는 것과 `withContext`를 사용하여 같은 코루틴 내에 존재하면서 코루틴의 context를 바꾼 것입니다.   

    [Ctx1 @coroutine#1] Started in ctx1
    [Ctx2 @coroutine#1] Working in ctx2
    [Ctx1 @coroutine#1] Back to ctx1   

이 예제는 코틀린 표준 라이브러리의 `use` 함수를 사용했습니다. `use` 함수는 `newSingleThreadContext`로 만들어진 threads가 더이상 필요하지 않을 때 release합니다.   

## Job in the context
코루틴의 Job은 context의 한 부분이며 `coroutineContext[Job]` 표현식을 사용하여 검색할 수 있습니다.
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")    
}
```   

debug mode에서 다음과 같은 출력문이 나옵니다.   
`My job is "coroutine#1:BlockingCoroutine{Active}@6d311334"`   

CoroutineScope의 isActive는 `coroutineContext[Job]?.isActive == true`의 짧은 버전입니다.


## Children of a coroutine
다른 코루틴의 CoroutineScope안에서 새로운 코루틴이 실행되면 CoroutineScope.coroutineContext를 통해 context를 상속받고 새로운 코루틴의 Job은 부모 코루틴 Job의 *자식*이 됩니다. 부모 코루틴이 취소되면 모든 자식 역시 재귀적으로 취소됩니다.

<br>

그러나, 부모-자식 관계는 다음 두 경우에 의해 재정의될 수 있습니다.

1. 코루틴이 실행될 때 다른 scope가 명백하게 지정되면(예를 들어, `GlobalScope.launch`) Job을 부모 scope로 부터 상속 받지 않습니다.
2. 다른 `Job` 객체가 새로운 코루틴의 context로 통과되면(아래 예시 참고) 부모 scope의 `Job`을 재정의합니다.

두 경우에 실행된 코루틴은 scope에 묶여있지 않습니다. 독립적으로 실행되고 작동합니다.

```kotlin
// launch a coroutine to process some kind of incoming request
val request = launch {
    // it spawns two other jobs
    launch(Job()) { 
        println("job1: I run in my own Job and execute independently!")
        delay(1000)
        println("job1: I am not affected by cancellation of the request")
    }
    // and the other inherits the parent context
    launch {
        delay(100)
        println("job2: I am a child of the request coroutine")
        delay(1000)
        println("job2: I will not execute this line if my parent request is cancelled")
    }
}
delay(500)
request.cancel() // cancel processing of the request
println("main: Who has survived request cancellation?")
delay(1000) // delay the main thread for a second to see what happens
```
    job1: I run in my own Job and execute independently!
    job2: I am a child of the request coroutine
    main: Who has survived request cancellation?
    job1: I am not affected by cancellation of the request

<br>

## Parental responsibilities
부모 코루틴은 항상 자식 코루틴이 전부 완료될 때까지 기다립니다. 부모 코루틴은 모든 자식 코루틴을 추적할 필요가 없고 자식들을 기다리기 위해 Job.join을 사용할 필요가 없습니다.   
```kotlin
// launch a coroutine to process some kind of incoming request
val request = launch {
    repeat(3) { i -> // launch a few children jobs
        launch  {
            delay((i + 1) * 200L) // variable delay 200ms, 400ms, 600ms
            println("Coroutine $i is done")
        }
    }
    println("request: I'm done and I don't explicitly join my children that are still active")
}
request.join() // wait for completion of the request, including all its children
println("Now processing of the request is complete")
```
    request: I'm done and I don't explicitly join my children that are still active
    Coroutine 0 is done
    Coroutine 1 is done
    Coroutine 2 is done
    Now processing of the request is complete

<br>

## Naming coroutines for debugging
코루틴이 자주 log를 남기고 단지 같은 코루틴에서 오는 밀접한 로그 기록을 볼 때, 자동으로 할당되는 id들은 유용합니다. 하지만, 코루틴이 구체적인 요구 사항에 묶여있거나 구체적인 background task에서 동작한다면, 디버깅을 위해 정확한 이름을 짓는게 좋습니다. CoroutineName context 요소는 thread name과 동일한 목표를 제공합니다. debugging mode가 켜졌을 때 코루틴을 실행하고 있는 thread name에 포함되어 있습니다.   

```kotlin
log("Started main coroutine")
// run two background value computations
val v1 = async(CoroutineName("v1coroutine")) {
    delay(500)
    log("Computing v1")
    252
}
val v2 = async(CoroutineName("v2coroutine")) {
    delay(1000)
    log("Computing v2")
    6
}
log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
```   

`-Dkotlinx.coroutines.debug` JVM option을 킨 출력 값은 다음과 같습니다.   

    [main @main#1] Started main coroutine
    [main @v1coroutine#2] Computing v1
    [main @v2coroutine#3] Computing v2
    [main @main#1] The answer for v1 / v2 = 42



## Combining context elements
때때로 coroutine context에 여러개의 요소를 정의할 필요가 있습니다. 이때 `+` 연산자를 사용할 수 있습니다. 예를들어, dispatcher과 이름을 동시에 지정할 수 있습니다.
```kotlin
launch(Dispatchers.Default + CoroutineName("test")) {
    println("I'm working in thread ${Thread.currentThread().name}")
}
```

`-Dkotlinx.coroutines.debug` JVM option을 킨 출력 값은 다음과 같습니다.    

     I'm working in thread DefaultDispatcher-worker-1 @test#2
<br>

## Coroutine scope
application이 lifecycle object를 가지고 있지만 object는 코루틴이 아니라고 가정해봅시다. 예를 들어 Android application을 작성하고 있습니다. 데이터를 가져오고 업데이트, 애니메이션 수행, 등을 하는 비동기 작업하려고 합니다. 이를 위해 다양한 코루틴을 Android activity context안에서 실행하고 있다고 해봅시다. 메모리 누출을 막기 위해 모든 코루틴들은 액티비티가 destoryed될 때 취소되야합니다. 물론 contexts와 jobs를 액티비티의 생명주기와 코루틴에 맞게 다루겠지만, **kotlinx.coroutines**는 CoroutineScope를 캡슐화하는 추상화를 제공합니다. 모든 코루틴 빌더가 코루틴의 확장으로 선언되므로 코루틴 scope에 대해 이미 잘 알고 있어야합니다.

<br>

액티비티의 생명주기와 얽매인 CoroutineScope 객체를 생성함으로써 코루틴의 생명주기를 관리할 수 있습니다. CoroutineScope() 또는 MainScope()의 factory functions를 이용하여 CoroutineScope를 생성할 수 있습니다. 전자의 경우 일반적인 목적의 scope이지만 후자의 경우 UI applications를 위한 scope를 만들며 Dispatchers.Main을 기본 dispatcher로 사용합니다.

```kotlin
class Activity {
    private val mainScope = MainScope()

    fun destroy() {
        mainScope.cancel()
    }
```

이제 scope가 정의된 액티비티의 scope안에서 코루틴을 실행할 수 있습니다. 

```kotlin
// class Activity continues
    fun doSomething() {
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            mainScope.launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
} // class Activity ends
```
main 함수에서 액티비티를 만들고 `doSomething` 함수를 호출합니다. 그리고 액티비티를 500ms 후에 destroy합니다. 이는 `doSomething`으로 시작된 모든 코루틴을 취소합니다. 더 이상 메세지가 print되지 않음을 통해 확인할 수 있습니다.
```kotlin
val activity = Activity()
activity.doSomething() // run test function
println("Launched coroutines")
delay(500L) // delay for half a second
println("Destroying activity!")
activity.destroy() // cancels all coroutines
delay(1000) // visually confirm that they don't work
```
    Launched coroutines
    Coroutine 0 is done
    Coroutine 1 is done
    Destroying activity!   

### Thread-local data
때때로 일부 thread-local 데이터를 코루틴으로 전달하거나 코루틴 사이에 전달하는 기능을 갖는 것이 편리합니다. 하지만 특정 thread에 bound되어 있지 않기 때문에 boilerplate를 유발할 수 있습니다.   

`ThreadLocal`, `asContextElement` extension 함수를 통해 해결할 수 있습니다. 코루틴이 context를 switch할때마다 주어진 `ThreadLocal` 값을 유지시키고 복구하는 추가적인 context element를 만듭니다.   

```kotlin
threadLocal.set("main")
println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
    println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    yield()
    println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
}
job.join()
println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
```   

Dispatchers.Default를 사용하여 background thread pool에서 새로운 코루틴을 실행하고 있습니다. 따라서 thread pool의 다른 스레드에서 작동하지만, thread local 변수 값을 가지고 있습니다. 이 변수 값은 `threadLocal.asContextElement(value = "launch")`를 통해 구체화되어 있습니다. 어느 thread에서 코루틴이 실행 중인지는 상관 없습니다. 따라서 debug 출력 결과는 다음과 같습니다.   

    Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
    Launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'launch'
    After yield, current thread: Thread[DefaultDispatcher-worker-2 @coroutine#2,5,main], thread local value: 'launch'
    Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'    

상응하는 context element를 설정해야한다는 것은 잊어버리기 쉽습니다. 만약 코루틴을 실행하는 스레드가 다르다면, 코루틴으로부터 접근된 thread-local 변수는 예상치 못한 값을 가질 수 있습니다. 이런 상황을 피하기 위해, ensurePresent 메서드를 사용하고 부적절한 사용에 대한 fail-fast를 사용하기를 추천합니다.   

`ThreadLocal`은 first-class support를 가지고 있으며 어떤 원시 `kotlinx.coroutines` provides와 사용할 수 있습니다. key를 하나만 가질 수 있지만, thread-local이 변경되었을 때 새로운 값은 코루틴 caller로 전달되지 않습니다. (context element는 모든 `ThreadLocal` 객체 접근을 모두 추적하지 못합니다.) 그리고 업데이트 된 값은 다음 suspension에서 유실됩니다. 코루틴에서 thread-local 값을 업데이트하기 위해 withContext를 사용하세요. [asContextElement](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/as-context-element.html)를 참고하세요.   

대안으로, 값은 `class Counter(var i : Int)` 처럼 mutale box에 저장될 수 있습니다. 이는 결국 thread-local 변수에 저장됩니다. 하지만 이 경우 mutable box안의 변수에 대한 잠재적인 동시 수정에 대한 동기화 책임이 있습니다.   

예를 들어 로깅 MDC, 트랜잭션 컨텍스트 또는 내부적으로 데이터 전달을 위해 스레드 로컬을 사용하는 기타 라이브러리와의 통합과 같은 고급 사용에 대해서는 구현해야 하는 [ThreadContextElement](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-thread-context-element/index.html) 인터페이스의 설명서를 참조하십시오.


