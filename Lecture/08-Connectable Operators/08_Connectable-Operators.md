# 08. Connectable Operators
- 연결 가능한 연산자~
- subscirbe할 때 요소의 emit을 시작하지 않고, `connect()` 메서드가 호출될 때만 요소를 내보내는 옵저버블 시퀀스 
  - 내가 선택한 시간에 요소를 방출하도록 옵저버블에 prompt할 수 있음
  - 이렇게 emit을 언제하느냐에 대한 시점 빼고는 일반 옵저버블 시퀀스와 동일함
- 이 방법으로, 모든 subscriber가 연결 가능한 옵저버블 시퀀스를 subscribe할 때까지 기다린 후에 요소를 emit할 수 있음

> 1. publish
> 2. replay
> 3. multicast

<br>

### 들어가기 전에) non-connectable 연산자 예시
- `interval` : 지정된 스케줄러에서 각 period가 지난 후에 요소를 내보내는 옵저버블 시퀀스를 만드는 오퍼레이터
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

sampleWithoutConnectableOperators()
```
- Subscirption 1이 찍히다가 5초 뒤 Subscription 2가 찍히기 시작
- Subscription 1과 2가 섞여서 찍힘
```
--- sampleWithoutConnectableOperators() example ---
Subscription: 1, Event: 0
Subscription: 1, Event: 1
Subscription: 1, Event: 2
Subscription: 1, Event: 3
Subscription: 1, Event: 4
Subscription: 1, Event: 5
Subscription: 2, Event: 0
Subscription: 1, Event: 6
Subscription: 2, Event: 1
Subscription: 1, Event: 7
Subscription: 2, Event: 2
```


<br>

## 1. publish
- 소스 옵저버블 시퀀스를 connectable 옵저버블 시퀀스로 변환하는 오퍼레이터
- coon
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
![image](https://user-images.githubusercontent.com/46644241/113640028-80dac500-96b5-11eb-942e-b3f3c59429eb.png)
```
--- sampleWithPublish() example ---
Subscription 1:, Event: 0
Subscription 1:, Event: 1
Subscription 2:, Event: 1
Subscription 1:, Event: 2
Subscription 2:, Event: 2
Subscription 1:, Event: 3
Subscription 2:, Event: 3
Subscription 3:, Event: 3

```
## 2. replay
- 초기 시드값 받아 옵저버블 시퀀스에서 내보낸 모든 항목에 Accumulator 클로저를 적용
- 집계결과는 single element 옵저버블 시퀀스로 반환
- 언제 구독하더라도(이미 요소를 방출하기 시작한 후라도) 동일한 element를 동일한 순서로 받아볼 수 있음
  - 모든 옵저버블이 같은 element을 같은 순서로 가지고 있음
  
![image](https://user-images.githubusercontent.com/46644241/113640113-b4b5ea80-96b5-11eb-9bd5-3d3f5741c461.png)

- If you apply the Replay operator to an Observable before you convert it into a connectable Observable, the resulting connectable Observable will always emit the same complete sequence to any future observers, even those observers that subscribe after the connectable Observable has begun to emit items to other subscribed observers.
- connectable 옵저버블로 변환하기 전에 replay 오퍼레이터를 적용하면, 결과적으로 connectable 옵저버블은 미래의 관찰자들에게 항상 same complete한 시퀀스를 emit함
  - 옵저버가 subscribe한 이후에 connectable 옵저버블이 다른 subscribed observers에게 item을 방출하는 것까지도 동일하게 처리
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
```
--- sampleWithReplayBuffer() example ---
Subscription 1:, Event: 0
Subscription 2:, Event: 0
Subscription 1:, Event: 1
Subscription 2:, Event: 1
Subscription 1:, Event: 2
Subscription 2:, Event: 2
Subscription 1:, Event: 3
Subscription 2:, Event: 3
Subscription 1:, Event: 4
Subscription 2:, Event: 4
Subscription 3:, Event: 0
Subscription 3:, Event: 1
Subscription 3:, Event: 2
Subscription 3:, Event: 3
Subscription 3:, Event: 4
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
```
--- sampleWithMulticast() example ---
Subject: 0
	Subscription 1:, Event: 0
Subject: 1
	Subscription 1:, Event: 1
	Subscription 2:, Event: 1
Subject: 2
	Subscription 1:, Event: 2
	Subscription 2:, Event: 2
Subject: 3
	Subscription 1:, Event: 3
	Subscription 2:, Event: 3
	Subscription 3:, Event: 3
Subject: 4
	Subscription 1:, Event: 4
	Subscription 2:, Event: 4
	Subscription 3:, Event: 4
Subject: 5
	Subscription 1:, Event: 5
	Subscription 2:, Event: 5
	Subscription 3:, Event: 5

```