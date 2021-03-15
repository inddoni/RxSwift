# 02. Creating and Subscribing to Observables
> There are several ways to create and subscribe to Observable sequences.
1. never
2. empty
3. just
4. of
5. from
6. create
7. range
8. repeatElement
9. generate
10. deferred
11. error
12. doOn


---

## 1. never
- 종료되지 않고, 어떤 이벤트도 생성하지 않는 Observable을 생성
- never emits any events(no items), never terminates
```swift
example("never") {
    let disposeBag = DisposeBag()
    let neverSequence = Observable<String>.never()
    
    let neverSequenceSubscription = neverSequence
        .subscribe { _ in
            print("This will never be printed")
    }
    
    neverSequenceSubscription.disposed(by: disposeBag)
}
```
<br>

## 2. empty
- Completed만 내보내는 빈 Observable을 생성
- no items, terminates normally
```swift
example("empty") {
    let disposeBag = DisposeBag()
    
    Observable<Int>.empty()
        .subscribe { event in
            print(event)
        }
        .disposed(by: disposeBag)
}
```
<br>


## 3. just
- single element를 갖는 Observable을 생성
- particular item을 가짐
```swift
example("just") {
    let disposeBag = DisposeBag()
    
    Observable.just("🔴")
        .subscribe { event in
            print(event)
        }
        .disposed(by: disposeBag)
}
```
<br>


## 4. of
- 고정된 수의 elements를 갖는 Observable을 생성
```swift
example("of") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐶", "🐱", "🐭", "🐹")
        .subscribe(onNext: { element in
            print(element)
        })
        .disposed(by: disposeBag)
}
```

- subscribes method
    - `subscribe(_:)` **an event handler / for all event types!** - Next, Error, and Completed) <br>
    - `subscribe(onNext:)` an element handler / only produce Next event elements (Ignore Error, Completed)
    - `subscribe(onError:)`
    - `subscribe(onCompleted:)`
    - `subscribe(onNext:onError:onCompleted:onDisposed:)` 
        - 하나 이상의 event 타입을 대응해야할 때
        - 어떤 이유로든 subscribe이 terminated되거나 disposed 될 때 
        ```swift
        someObservable.subscribe(
            onNext: { print("Element:", $0) },
            onError: { print("Error:", $0) },
            onCompleted: { print("Completed") },
            onDisposed: { print("Disposed") }
        )
        ```

## 5. from
- Array, Dictionary, Set 같은 Sequence로 Observable을 생성
```swift

```
<br>

## 6. create
- custom Observable을 생성
- observer를 parameter로 갖는 function으로 만듬 (observer의 onNext, onError, onCompleted methods를 적절히 호출)
- well-formed Oberservable (finite)
    - observer의 `onCompleted` method를 정확히 한번만 호출하거나
    - observer의 `onError` method를 정확히 한번만 호출하려고 시도해야함
    - 그리고 그 후에는 observer의 어떤 method도 호출해서는 안됨


```swift
example("create") {
    let disposeBag = DisposeBag()
    
    let myJust = { (element: String) -> Observable<String> in
        return Observable.create { observer in
            observer.on(.next(element))
            observer.on(.completed)
            return Disposables.create()
        }
    }
        
    myJust("🔴")
        .subscribe { print($0) }
        .disposed(by: disposeBag)
}
```
<br>


## 7. range
- 일련의 정수 범위를 emits 후 종료되는 Observable을 생성
- 범위의 시작과 length를 주고 sequential한 정수 범위를 emits
```swift
example("range") {
    let disposeBag = DisposeBag()
    
    Observable.range(start: 1, count: 10)
        .subscribe { print($0) }
        .disposed(by: disposeBag)
}
```
<br>

## 8. repeatElement
- 주어진 elements를 무기한으로(indefinitely) emits하는 Observable 생성
- 특정 항목(particular item)을 multiple times 생성
- sequence item 반복, 반복 횟수 제한 가능
- `take()` - 지정된 수만큼 elements를 return해줌
```swift
example("repeatElement") {
    let disposeBag = DisposeBag()
    
    Observable.repeatElement("🔴")
        .take(3)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
<br>

## 9. generate
- condition이 `true`일 동안에 값을 생성하는 Observable 생성

```swift
example("generate") {
    let disposeBag = DisposeBag()
    
    Observable.generate(
            initialState: 0,
            condition: { $0 < 3 },
            iterate: { $0 + 1 }
        )
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
<br>

## 10. deferred
- 각 subscriber에 대한 Observable 생성 
- observer가 subscribe할 때까지 Observable을 생성하지 않음 
- subscribe하면 일반적으로 Observable fuctory function을 사용하여 생성함
- 각 subscriber별로 위 작업을 새로 수행 (각각 따로 수행될 수 밖에 없음)
- 동일한 Observable을 subscribing 한다고 생각할 수 있지만 각 subscriber는 고유한 개별(own individual) 시퀀스를 얻음
- 경우에 따라, Observable을 생성하기 위해 마지막까지(subscrption time) 기다리면 Observable에 최신 데이터가 포함될 수 있음

```swift
example("deferred") {
    let disposeBag = DisposeBag()
    var count = 1
    
    let deferredSequence = Observable<String>.deferred {
        print("Creating \(count)")
        count += 1
        
        return Observable.create { observer in
            print("Emitting...")
            observer.onNext("🐶")
            observer.onNext("🐱")
            observer.onNext("🐵")
            return Disposables.create()
        }
    }
    
    deferredSequence
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    deferredSequence
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
<br>

## 11. error
- emits되는 아이템이 없고, 에러로 즉시 종료되는 Observable 생성
```swift
example("error") {
    let disposeBag = DisposeBag()
        
    Observable<Int>.error(TestError.test)
        .subscribe { print($0) }
        .disposed(by: disposeBag)
}
```
<br>

## 12. doOn
- 각 emitted event들에 대해 side-effect action을 호출하고
- 원래 이벤트를 return 하도록 하는 것 (original event passes through)
- 다양한 Observable lifecycle 이벤트에 대해 각 수행할 작업을 등록
- Observable에서 특정한 이벤트가 발생할 때, ReactiveX에서 호출할 callback을 등록할 수 있음
    - 이 callback은 Observable에 종속된 일반 notifications set과는 별도로 호출됨 (independently함!)
- 특정 이벤트를 intercept하는 methods 
    - `doOnNext(_:)`, `doOnError(_:)`, `doOnCompleted(_:)`
- 단일 호출로, 하나 이상의 이벤트를 intercept하는 method
    - `doOn(onNext:onError:onCompleted:)`
```swift
example("doOn") {
    let disposeBag = DisposeBag()
    
    Observable.of("🍎", "🍐", "🍊", "🍋")
        .do(onNext: { print("Intercepted:", $0) }, afterNext: { print("Intercepted after:", $0) }, onError: { print("Intercepted error:", $0) }, afterError: { print("Intercepted after error:", $0) }, onCompleted: { print("Completed")  }, afterCompleted: { print("After completed")  })
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
<br>
