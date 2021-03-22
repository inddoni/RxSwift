# 03. Working with Subjects
> 
1. Subject란?
2. PublishSubject
3. ReplaySubject
4. BehaviorSubject


---

## 1. Subject란?
- Rx의 일부 implementation에서 사용할 수 있는 일종의 bridge 또는 proxy
- observer와 Observable처럼 사용 가능 (두 가지 역할을 모두 수행)
  - observer ~> 하나 이상의 Observable을 subscribe할 수 있고
  - Observable ~> 새로운 item emit하고, 관찰한 item을 reemit하여 통과할 수 있음

## 2. PublishSubject
> subscription 시간을 기준으로 모든 observer에게 새로운 이벤트를 broadcast 할 수 있음
<br>

<img width="640" alt="S PublishSubject" src="https://user-images.githubusercontent.com/46644241/112066787-54e61c80-8baa-11eb-9f5b-c6dc0ceaef28.png">

```swift
example("PublishSubject") {
    let disposeBag = DisposeBag()
    let subject = PublishSubject<String>()
    
    subject.addObserver("1").disposed(by: disposeBag)
    subject.onNext("🐶")
    subject.onNext("🐱")
    
    subject.addObserver("2").disposed(by: disposeBag)
    subject.onNext("🅰️")
    subject.onNext("🅱️")
}
```
<br>

## 3. ReplaySubject
> 모든 subscribers에게 새 이벤트를 broadcast 해주면서, <br>
> 새 subscriber에게 이전에 발생한 이벤트의 bufferSize를 내려줌
<br>

<img width="640" alt="S ReplaySubject" src="https://user-images.githubusercontent.com/46644241/112066777-51529580-8baa-11eb-9304-4168d081b7f9.png">

```swift
example("ReplaySubject") {
    let disposeBag = DisposeBag()
    let subject = ReplaySubject<String>.create(bufferSize: 1)
    
    subject.addObserver("1").disposed(by: disposeBag)
    subject.onNext("🐶")
    subject.onNext("🐱")
    
    subject.addObserver("2").disposed(by: disposeBag)
    subject.onNext("🅰️")
    subject.onNext("🅱️")
}
```
<br>

## 4. BehaviorSubject
> 모든 subscribers에게 새 이벤트를 broadcast 해주면서, <br>
> 새 subscriber에게 가장 최근에 발생한 이벤트의 값 (혹은 initial value)를 내려줌

<br>

<img width="640" alt="S BehaviorSubject" src="https://user-images.githubusercontent.com/46644241/112066782-53b4ef80-8baa-11eb-93de-2180785f4f4b.png">

```swift
example("BehaviorSubject") {
    let disposeBag = DisposeBag()
    let subject = BehaviorSubject(value: "🔴")
    
    subject.addObserver("1").disposed(by: disposeBag)
    subject.onNext("🐶")
    subject.onNext("🐱")
    
    subject.addObserver("2").disposed(by: disposeBag)
    subject.onNext("🅰️")
    subject.onNext("🅱️")
    
    subject.addObserver("3").disposed(by: disposeBag)
    subject.onNext("🍐")
    subject.onNext("🍊")
}
```
<br>

## 5. Subject의 Completed Event
- disposed 되려고 할때나 completed 되면 이 subject의 observable에서 더이상 이벤트를 자동으로 emits하지 않음
  - error로 종료되면 subscribe 할 때, error를 emit해주고
  - completed로 동료되면 subscriber 할 때, completed를 emit해줌


## Reference
- ReactiveX 공식 홈페이지의 [Subject Docs](http://reactivex.io/documentation/subject.html)