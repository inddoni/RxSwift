# 01. Introduction

1. 왜 RxSwift를 쓸까?
2. Concepts
3. Observable Sequence
4. Observables and observers (aka subscribers)
- Appendix) Installing RxSwift (Using CocoaPods)

---
## 1. 왜 RxSwift를 쓸까?
> **Code의 대부분이 외부 Event에 대한 Response 처리**

    - @IBAction handler를 쓰거나
    - 키보드 변화 등에 대한 notification을 observation하거나
    - URL session이 response를 내려줄 때 실행할 Clouser를 정의하거나
    - Variable의 변화를 감지하기 위해 KVO를 사용하거나

⇒ 이런 것들 때문에 너무 불필요하게 복잡해짐 🤯 <br>
**⇒ 모든 Call & Response를 handling하는 하나의 일관된 시스템 "RX"**
=> Rxswif는 Observable과 functional style operator를 제공하여 비동기 및 event-based 코드를 작성하도록 하는 라이브러리

## 2. Concepts
> **Every Observable instance is just a sequence.**
* Observable의 가장 큰 장점
    - elements를 비동기적으로 받을 수 있다!
- Can receive elements asyncronously.
    - 이것이 Rxswift의 본질
    - 모든 것은 여기서부터 개념을 확장해나간 것

## 3. Observable Sequence
- `Event.next(Element)`
    - Observable이 다음 event를 생성하고 이런식으로 더 많은 이벤트를 계속 생성할 수 있음
    - 이전 event가 완료되기 전에 다음 event를 생성(전송)
- `Event.error(ErrorType)` or `Event.completed`
    - 에러나 완료된 이벤트의 경우 subscriber에게 추가 이벤트를 생성할 수 없음
- Sequence grammar
    - `next* (error | completed)?`

## 4. Observables and observers (aka subscribers)
- Observable은 subscriber가 없으면 subscription closure를 실행하지 않음
```swift
example("Observable with no subscribers") {
    _ = Observable<String>.create { observerOfString -> Disposable in
        print("This will never be printed")
        observerOfString.on(.next("😬"))
        observerOfString.on(.completed)
        return Disposables.create()
    }
} 
```
- Subscribe(_: )가 호출되서 실행되는 Closure
```swift
example("Observable with subscriber") {
  _ = Observable<String>.create { observerOfString in
            print("Observable created")
            observerOfString.on(.next("😉"))
            observerOfString.on(.completed)
            return Disposables.create()
        }
        .subscribe { event in
            print(event)
    }
}
```

## Appendix. Installing RxSwift (Using CocoaPods)
```vim
# Podfile
use_frameworks!

target 'YOUR_TARGET_NAME' do
    pod 'RxSwift', '6.1.0'
    pod 'RxCocoa', '6.1.0'
end

# RxTest and RxBlocking make the most sense in the context of unit/integration tests
target 'YOUR_TESTING_TARGET' do
    pod 'RxBlocking', '6.1.0'
    pod 'RxTest', '6.1.0'
end

```