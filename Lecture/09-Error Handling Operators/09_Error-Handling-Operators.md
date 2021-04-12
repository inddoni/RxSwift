# 09. Error Handling Operators
> 옵저버블에서 보낸 error notification을 복구하는데 도움을 주는 연산자

1. catchErrorJustReturn
2. catchError
3. retry
4. retry(_:)

## 0. catch
![image](https://user-images.githubusercontent.com/46644241/114401276-ba439100-9bdd-11eb-8d20-b52b5981dd05.png)
- 오류 없이 시퀀스를 계속하여 `onError` notification을 복구
- 소스 옵저버블에서 `onError` notification을 intercept 하고, 이를 옵저버에게 전달하는 대신 다른 item이나 item 시퀀스로 대체하여 정상적으로 종료하거나 전체적으로 종료되지 않도록 잠재적으로 허용함

## 1. catchErrorJustReturn
- single element를 내보내고 종료되는 옵저버블 시퀀스를 반환하여 error 이벤트를 복구하는 오퍼레이터
```swift
example("catchErrorJustReturn") {
    let disposeBag = DisposeBag()
    
    let sequenceThatFails = PublishSubject<String>()
    
    sequenceThatFails
        .catchErrorJustReturn("😊") //subscribe 전에
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
    sequenceThatFails.onNext("😬")
    sequenceThatFails.onNext("😨")
    sequenceThatFails.onNext("😡")
    sequenceThatFails.onNext("🔴")
    sequenceThatFails.onError(TestError.test)
}
```
```
--- catchErrorJustReturn example ---
next(😬)
next(😨)
next(😡)
next(🔴) //원래 여기서 error(test) 뜨고 끝
next(😊) 
completed //error 안뜨고 completed 됨
```
## 2. catchError
- 제공된 복구 Observable 시퀀스로 전환하여 오류 이벤트에서 복구합니다.
- return AObservableSequence 하면 error 발생했을 때 AObservableSequence가 이어서 일하기 시작 
- Q. 이어서(전환해서) 오류 이벤트를 복구한다는 것의 활용?
```swift
example("catchError") {
    let disposeBag = DisposeBag()
    
    let sequenceThatFails = PublishSubject<String>()
    let recoverySequence = PublishSubject<String>()
    
    sequenceThatFails
        .catchError {
            print("Error:", $0)
            return recoverySequence
        }
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
    sequenceThatFails.onNext("😬")
    sequenceThatFails.onNext("😨")
    sequenceThatFails.onNext("😡")
    sequenceThatFails.onNext("🔴")
    sequenceThatFails.onError(TestError.test) // 여기서 sequenceThatFails는 죽음
    
    recoverySequence.onNext("😊") // sequenceThatFails 시퀀스를 subscribe한 것에서 emit 됨
}
```
```
--- catchError example ---
next(😬)
next(😨)
next(😡)
next(🔴)
Error: test
next(😊)
```

## 3. retry
- Observable sequence를 무기한으로 재구독해서 반복적으로 error events를 복구하도록 하는 오퍼레이터
- Retry 연산자는 error notification을 관찰자에게 전달하지 않고, 대신 소스 Observable을 resubscribe하고 오류없이 시퀀스를 complete 할 수있는 또 다른 기회를 제공함.
![image](https://user-images.githubusercontent.com/46644241/114401123-96804b00-9bdd-11eb-99b5-836676882847.png)
- `Retry`는 항상 onNext notification을 observer에게 전달하므로(오류로 종료되는 시퀀스에서도!!) 중복 방출이 발생할 수 있음
```swift
example("retry") {
    let disposeBag = DisposeBag()
    var count = 1
    
    let sequenceThatErrors = Observable<String>.create { observer in
        observer.onNext("🍎")
        observer.onNext("🍐")
        observer.onNext("🍊")
        
        if count == 1 {
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
        .retry()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
```
--- retry example ---
🍎
🍐
🍊
Error encountered
🍎
🍐
🍊
🐶
🐱
🐭
```

## 4. retry(_:)
- 최대 재시도 횟수 `maxAttemptCount`까지 Observable 시퀀스를 resubscribe하여 error event를 반복적으로 복구합니다.
```swift
example("retry maxAttemptCount") {
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
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
```
--- retry maxAttemptCount example ---
🍎
🍐
🍊
Error encountered
🍎
🍐
🍊
Error encountered
🍎
🍐
🍊
Error encountered
Unhandled error happened: test
```