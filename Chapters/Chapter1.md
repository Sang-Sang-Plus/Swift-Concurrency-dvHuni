# Why Modern Swift Concurrency

Apple은 **asynchronous**(비동기성) framework인 **GCD**라는 개념을 만들었고, 현재까지도 **concurrency**(병행성)와 **asynchrony**에 많은 도움을 주고있습니다.

하지만 Swift 5.5에서 새로운 asynchronous, concurrent code를 소개하면서 모든것을 바꾸었습니다.

새로운 concurrency model은 Swift에서 더 안전하고 효과적인 프로그램을 작성할 수 있도록 다음과 같은 방법들이 포함되었습니다.

- 구조화된 방법으로 asynchronous 작업을 수행하는 새로운 native syntax
- asynchronous를 설계하고, concurrent code를 작성하는 표준 API bundle


## Understanding asynchronous and concurrent code

기존의 code들은 **top-bottom** 식으로 작성했고, **line-by-line**으로 함수가 시작되고 종료되었습니다.

이것은 code line의 실행을 결정하기 쉽게 만들었고, 따르기도 간단했습니다.

함수를 호출하는것도 마찬가지입니다. 

**code는 synchronously(동시적)으로 동작할때, 실행은 sequentially(순차적)으로 일어납니다.**

synchronous context에서 code는 signle CPU core일 때, 하나의 excution thread에서 동작합니다.

이는 일방통행인 길을 생각하면 이해가 쉽습니다.
</br>


급하게 길을 지나가야 하는 구급차(high priority)라도 앞의 차가 지나가지 않는다면 지나갈 수 없습니다.

iOS app과 Cocoa-base인 mac OS app은 inherently asynchronous(본질적인 비동기)입니다.

**Asynchronous excutution(비동기 실행)은 하나의 thread를 여러 임의 조각으로 나누어 프로그램을 실행** 할 수 있게 합니다.

사용자 입력, 네트워크 연결 등 다양한 이벤트를 여러 thread에서 동시에 실행 할 수 있습니다.
</br>


synchronous context와 다르게 asynchronous context는 asychronous function들이 동일한 thread에서 실행될때는 fucntion에 대한 정확한 순서를 말하기 어렵습니다.

![image](https://user-images.githubusercontent.com/39300449/166091286-01e79299-b0a5-4706-b76f-b4d4dc966c80.png)

> *위 그림의 왼쪽 도로처럼 빨간 차가 우선적으로 가야할지, 녹색 차가 우선적으로 가야할지 기준이 없다면 정확한 순서를 말하기 어렵다.*

</br>


실 예시는 다음과 같은 예시를 들 수 있습니다.
![image](https://user-images.githubusercontent.com/39300449/166091294-eb4ef49d-fcdc-427c-ab50-6431d0432179.png)

Netwroking Request가 서버측의 응답을 받고 실행되는 completion closure를 제공할 때 completion의 실행을 기다리는 동안 사용자는 다른 동작을 수행할 수 있습니다.

</br>

의도적으로 프로그램의 일부분을 **병렬적으로** 실행시키려면 **concurrent** API를 사용해야 합니다.

이때 몇몇 API들은 한번에 고정된 수의 task를 실행하도록 지원하고, 다른 API들은 고정되지 않은 수의 concurrent taks를 실행 할 수 있도록 지원합니다.

이것은 무수한 concurrency-related problem을 일으킵니다.

예를들어 프로그램의 다른 부분에서 동시에 동일한 variable에 접근한다면 앱이 죽거나, 기대되지않은 app state 오류를 일으킬 수 있습니다.

하지만 concurrency를 적절하게 사용한다면 프로그램은 동시에 다른 function들을 multi CPU core에서 실행 시키기 때문에 더 빠르게 동작할 수 있습니다.

![image](https://user-images.githubusercontent.com/39300449/166091356-13e156ea-b374-45df-80f3-27927a87c1fd.png)

> *여러 차선이 있는 도로와 같으며 우선순위(priority)에 따라서 실행 순서가 변경될 수 있습니다.*

</br>

마찬가지로 실제 예시를 살펴보겠습니다.

서버에서 이미지 그룹을 다운로드 받고, 동시에 thumbnail 사이즈로 scale down, chache에 저장하는 기능을 하는 photo browsing app은 다음과 같이 동작할 수 있습니다.

![image](https://user-images.githubusercontent.com/39300449/166091372-565829a3-6c4d-45e0-bbdb-a17528ee43cf.png)

> *각 thread의 기능 block이 각기 위치가 다른 이유가 있습니다.
무엇 때문일까요?*

asynchrony와 concurrency는 듣기에는 좋아보입니다.

또한 이미Apple에서는 GCD를 통해 제공하고 있었습니다.

그렇다면 왜 Swift는 새로운 concurrency model이 필요했을까요?
</br></br>

### Reviewing the existing concurrency options

GCD에서는 dispatch queue라는 추상화된 thread를 통해 asynchronus code를 작동시켰습니다.

이러한 API는 모두 POSIX threads라는 동일한 foundation을 사용하고 있습니다.

POSIX thread는 프로그래밍 언어에 구애받지 않는 표준화된 execution model입니다. 각각의 excution flow는 thread이며, multiple thread는 이전에 보았던 여러 차선이 있는 도로와 비슷합니다.

Thread wrapper인 Operation과 Thread는 수동으로 execution을 관리해야 합니다. 다시 말해 thread를 생성, 파괴, 실행 우선순위를 부여할 책임 그리고 서로 다른 thread에서 공유되는 데이터에 대한 synchronizing에 대한 책임은 당신에게 있습니다.

GCD의 queue-based model은 잘 동작하지만 다음과 같은 문제를 일으킬 수 있습니다.

- Thread explosion: active thread와 지속적으로 스위칭하는 concurreny thread를 너무 많이 생성한 경우, 앱이 느려질 수 있습니다.
- Priority inversion: 임의적으로 동일한 queue에서 낮은 우선순위의 task가 높은 우선순위의 task를 block 하는 경우가 발생합니다.
- Lack of executon hierarchy: asynchronus code block은 각각의 task를 독립적으로 관리하게되고, 이는 실행중인 task에 대해 접근, 실행 취소를 어렵게 만듭니다. 또한 중첩된 hierarchy는 결과의 return을 복잡하게 만듭니다.
</br></br>

### Introducing the modern Swift concurrency model

새로운 concurrency model은 Swift, Xcode, runtime에 밀접하게 통합되어 있습니다.

또한 다음 다섯가지의 new key feature를 가지고 있습니다.

1. A cooperative thread pool
    
    새로운 model은 thread pool을 CPU core 수보다 많지 않게 관리합니다. 이는 런타임에서 thread를 생성, 파괴하는 관리가 필요 없음을 의미하며, thread를 교환하는 작업을 실행하지 않아도 된다는 의미입니다.
    
2. async/awiat syntax
    
    Swift의 새로운 async/await 구문은 compiler가 해당 code가 suspend, resume 될 수 있음을 나타냅니다. 추가로 해당 구문은 escaping closure를 사용하지 않기 때문에 weak, strong caputre를 사용하지 않는 경우가 있습니다.
    
3. Structured concurrency
    
    asynchronous task는 hierarchy의 일부분이며, parent task가 하위 task execution에 대한 priority를 제공합니다. 이러한 구조는 parent task가 중지되었을 때, 하위 task에 대한 중지를 가능하게 합니다. 또한 parent task가 완료되기 전 까지 모든 하위태스크가 완료되기까지 기다릴 수 있습니다.
    
    이러한 hierarchy는 high-priority의 task가 low-priority task보다 먼저 실행됨을 보장합니다.
    
4. Context-aware code compilation
    
    컴파일러는 code가 비동기적으로 실행 될 수 있는지 추적합니다. 이에 따라 당신은 적절하지 않은 코드를 작성할 수 없게 됩니다. 이러한 컴파일러의 인식은 actor 라는 새로운 기능을 통해 가능합니다.
</br></br>

## Writing your first async/await

책의 내용을 따라가다보면 샘플 프로젝트를 만나게됩니다.

샘플 프로젝트는 직접 따라해보진 않고, 코드에 대한 설명을 정리해보겠습니다.

서버로부터 JSON을 가져오는 작업에 대한 코드를 작성하는 과정에서 modern-concurrency를 사용합니다.

```swift
func availableSymbols() async throws -> [String] {
  guard let url = URL(string: "http://localhost:8080/littlejohn/symbols") 
  else {
    throw "The URL could not be created."
  }
}
```

`async` 키워드는 컴파일러에게 해당 코드가 asynchronous context로 동작하도록 알려줍니다. 다시말해 해당 코드는 **suspend, resume** 될 수 있다는 것을 의미합니다.

method가 완료되는데 얼마가 걸리든 상관없이 궁극적으로 해당 코드는 synchronous method 처럼 값을 return합니다.

다음으로는 method에 아래 코드를 추가합니다.

```swift
let (data, response) = try await URLSession.shared.data(from: url)
```

```swift
func availableSymbols() async throws -> [String] {
  guard let url = URL(string: "http://localhost:8080/littlejohn/symbols") 
  else {
    throw "The URL could not be created."
  }

	let (data, response) = try await URLSession.shared.data(from: url)
}
```

만들어진 url을 통해 set을 가져옵니다.

위 코드의 동작은 다음과 같습니다.

async method인 availableSymbols()은 suspend 될수 있으며, URLSession.shared.data(from: delegate:) method에서 data를 가져와야 resume 됩니다.

![image](https://user-images.githubusercontent.com/39300449/166091503-d6cadea8-0bd4-4d8f-8695-95e57641f3ba.png)

`await` 키워드는 suspend 될 point를 지정하는 keyword 입니다.

이것이 바로 당신이 asynchronous call을 사용하면서도 thread와 closure를 신경써도 되지 않는 이유입니다.

다음으로는 data를 가져온 뒤, decode하는 작업이 필요합니다.

```swift
guard (response as? HTTPURLResponse)?.statusCode == 200 else {
  throw "The server responded with an error."
}

return try JSONDecoder().decode([String].self, from: data)
```

```swift
func availableSymbols() async throws -> [String] {
  guard let url = URL(string: "http://localhost:8080/littlejohn/symbols") 
  else {
    throw "The URL could not be created."
  }

	let (data, response) = try await URLSession.shared.data(from: url)

	guard (response as? HTTPURLResponse)?.statusCode == 200 else {
	  throw "The server responded with an error."
	}

	return try JSONDecoder().decode([String].self, from: data)
}
```

이렇게 첫번째 async-await에 대한 사용은 마무리 되었습니다. 
</br></br>

### Using async/await in SwiftUI & UIKit

다음은 이전에 만들어진 함수를 SwiftUI과 UIKit 환경에서 활용하는 방법입니다.

책에서는 SwiftUI를 기준으로 설명하지만 UIKit 환경에서도 동일하게 사용할 수 있습니다.

뷰 혹은 화면이 나타났을 때 작성된 함수를 통해 아래처럼 데이터를 가져오려고 합니다.

```swift
// SwiftUI
.onAppear {
	try await model.availableSymbols()
}

// UIKit
override func viewDidAppear() {
	super.viewDidAppear()
	try await model.availableSymbols()
}
```

![image](https://user-images.githubusercontent.com/39300449/166091496-68707f57-b643-43a2-b24c-0879b37243e4.png)

하지만 위 코드를 작성하려고 하면 자동완성에서는 에러를 뱉고있고, 실제로 작성하게 되면 에러가 납니다.

```swift
Invalid conversion from throwing function of type '() async throws -> Void' to non-throwing function type '() -> Void

```

에러가 발생하는 이유는 onAppear, viewDidAppear가 synchronous 이기 때문에, concurrenct context가 아닌 함수에서 asynchronous code를 사용하려 하기 때문입니다.

해당 이슈를 해결하려면, .task, Task 를 사용하여 asynchronous code를 사용하는것을 다음과 같이 명시해야 합니다.

```swift
// SwiftUI
.task {
	try await model.availableSymbols()
}

// UIKit
override func viewDidAppear() {
	super.viewDidAppear()
	Task {
		try await model.availableSymbols()	
	}
}
```

availableSymbols()는 `throws`키워드를 통해 Error를 방출 할 수 있기 때문에 catch를 작성해야합니다.

```swift
// SwiftUI
.task {
	do {
		let symbol = try await model.availableSymbols()
	} catch {
		// error handling
	}
}

// UIKit
override func viewDidAppear() {
	super.viewDidAppear()
	Task {
		do {
				try await model.availableSymbols()	
		} catch {
			// error handling
		}
	}
}
```
</br></br>

## Canceling tasks in structured concurrency

concurrent code는 구조적인 방법으로 실행되며, hierarchy내에서 엄격하게 Task들은 동작합니다. 따라서 **runtime에서 특정 task의 parent를 알 수 있고**, **어떤 tasks들이 inhreit 될지 알 수 있습니다.**

![image](https://user-images.githubusercontent.com/39300449/166091491-1d3b03e6-9f8a-4482-8a20-2a870d0f411e.png)

예를 들어 위 사진을 보면

.task는 비동기적으로 함수를 호출하고, 해당 함수는 비동기적으로 AsyncSequence를 반환하는 URLSession.bytes의 결과를 await 합니다.

각각의 suspension point는 `await` 키워드에 의해 이루어지며, 이 때 마다 thread는 변화할 수 있습니다.

**task에 의해 실행된 async task는 excution thread 혹은 suspension state에 관계없이 다른 모든 task들의 parent이기 때문**입니다.

SwiftUI에서의 task modifier는 view가 사라졌을 때 asynchronous code를 취소하는것에 주의해야합니다.

또한 모든 asynchronous tasks는 user의 navigate에 의해 화면을 벗어날 때 취소되는것에 주의해야합니다. (= UIKit에서 사용되는 Task도 동일하다는 의미입니다.)

![image](https://user-images.githubusercontent.com/39300449/166091487-3edcc796-aaaa-4c4f-b6e4-37aeb4f5249f.png)
</br></br>

# Key points

- Swift 5.5에서는 기존에 존재하던 concurrency issues(thread explosion, priority inversion)들을 해결한 새로운 concurrency model을 소개했습니다.
- `async` 키워드를 통해 function이 asynchronous임을 나타내며, `await` 키워드를 통해 asynchronous function의 결과를 비동기적으로 기다릴 수 있습니다.
- 시간의 흐름에 따라 asynchronous sequence를 for-try-await loop 구문을 통해 반복 할 수 있습니다


> 사용된 이미지들은 [Modern Concurrency in Swift](https://www.raywenderlich.com/books/modern-concurrency-in-swift)에 가져온 것임을 밝힙니다.
