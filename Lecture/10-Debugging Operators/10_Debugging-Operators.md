# 10. Debugging-Operators
> Rx code의 디버깅을 돕는 오퍼레이터
1. debug
2. RxSwift.Resources.total

## 1. debug
- 모든 subscriptions, events, disposals를 프린트함
```swift
example("debug") {
    let disposeBag = DisposeBag()
    var count = 1
    
    let sequenceThatErrors = Observable<String>.create { observer in
        observer.onNext("🍎")
        observer.onNext("🍐")
        observer.onNext("🍊")
        
        if count < 5 {
            observer.onError(TestError.test)
            print("Error encountered")
            count += 1
        }
        
        observer.onNext("🐶")
        observer.onNext("🐱")
        observer.onNext("🐭")
        observer.onCompleted()
        
        return Disposables.create()
    }
    
    sequenceThatErrors
        .retry(3)
        .debug()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```

## 2. RxSwift.Resources.total
- 모든 Rx 리소스 할당 수를 제공하여 개발 중에 누수를 감지하는 데 유용하게 사용!
```swift
#if NOT_IN_PLAYGROUND
#else
example("RxSwift.Resources.total") {
    print(RxSwift.Resources.total) // 1
    
    let disposeBag = DisposeBag()
    
    print(RxSwift.Resources.total) // 3 let 선언
    
    let subject = BehaviorSubject(value: "🍎")
    
    let subscription1 = subject.subscribe(onNext: { print($0) }) // 🍎
    
    print(RxSwift.Resources.total) // 10 
    
    let subscription2 = subject.subscribe(onNext: { print($0) }) // 🍎
    
    print(RxSwift.Resources.total) // 13
    
    subscription1.dispose() 
    
    print(RxSwift.Resources.total) // 11
    
    subscription2.dispose()
    
    print(RxSwift.Resources.total) // 9
}
    
print(RxSwift.Resources.total) // 1
#endif
```
```
1
3
🍎
10
🍎
13
11
9
1
```
- 왜 2개씩 늘고 줄음,,?