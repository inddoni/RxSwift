# 08. Connectable Operators
> 
```swift
func sampleWithoutConnectableOperators() {
    printExampleHeader(#function)
    
    let interval = Observable<Int>.interval(.seconds(1), scheduler: MainScheduler.instance)
    
    _ = interval
        .subscribe(onNext: { print("Subscription: 1, Event: \($0)") })
    
    delay(5) {
        _ = interval
            .subscribe(onNext: { print("Subscription: 2, Event: \($0)") })
    }
}
```
1. publish
2. replay
3. multicast
<br>
<br>

## 1. publish
- 

```swift
func sampleWithPublish() {
    printExampleHeader(#function)
    
    let intSequence = Observable<Int>.interval(.seconds(1), scheduler: MainScheduler.instance)
        .publish()
    
    _ = intSequence
        .subscribe(onNext: { print("Subscription 1:, Event: \($0)") })
    
    delay(2) { _ = intSequence.connect() }
    
    delay(4) {
        _ = intSequence
            .subscribe(onNext: { print("Subscription 2:, Event: \($0)") })
    }
    
    delay(6) {
        _ = intSequence
            .subscribe(onNext: { print("Subscription 3:, Event: \($0)") })
    }
}

```
## 2. replay
- 초기 시드값 받아 옵저버블 시퀀스에서 내보낸 모든 항목에 Accumulator 클로저를 적용
- 집계결과는 single element 옵저버블 시퀀스로 반환
```swift
func sampleWithReplayBuffer() {
    printExampleHeader(#function)
    
    let intSequence = Observable<Int>.interval(.seconds(1), scheduler: MainScheduler.instance)
        .replay(5)
    
    _ = intSequence
        .subscribe(onNext: { print("Subscription 1:, Event: \($0)") })
    
    delay(2) { _ = intSequence.connect() }
    
    delay(4) {
        _ = intSequence
            .subscribe(onNext: { print("Subscription 2:, Event: \($0)") })
    }
    
    delay(8) {
        _ = intSequence
            .subscribe(onNext: { print("Subscription 3:, Event: \($0)") })
    }
}
```
## 3. multicast
- 소스 옵저버블 시퀀스를 connectable 시퀀스로 변환
- 특정 subject를 통해 emit하는 것을 브로드 캐스트 함
```swift
func sampleWithMulticast() {
    printExampleHeader(#function)
    
    let subject = PublishSubject<Int>()
    
    _ = subject
        .subscribe(onNext: { print("Subject: \($0)") })
    
    let intSequence = Observable<Int>.interval(.seconds(1), scheduler: MainScheduler.instance)
        .multicast(subject)
    
    _ = intSequence
        .subscribe(onNext: { print("\tSubscription 1:, Event: \($0)") })
    
    delay(2) { _ = intSequence.connect() }
    
    delay(4) {
        _ = intSequence
            .subscribe(onNext: { print("\tSubscription 2:, Event: \($0)") })
    }
    
    delay(6) {
        _ = intSequence
            .subscribe(onNext: { print("\tSubscription 3:, Event: \($0)") })
    }
}
```