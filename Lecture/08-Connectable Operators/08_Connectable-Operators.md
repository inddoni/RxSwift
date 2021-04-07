# 08. Connectable Operators
- 연결 가능한 연산자~
- subscirbe할 때 요소의 emit을 시작하지 않고, `connect()` 메서드가 호출될 때만 요소를 내보내는 옵저버블 시퀀스 
  - 내가 선택한 시간에 요소를 방출하도록 옵저버블에 prompt할 수 있음
  - 이렇게 emit을 언제하느냐에 대한 시점 빼고는 일반 옵저버블 시퀀스와 동일함
- 이 방법으로, 모든 subscriber가 연결 가능한 옵저버블 시퀀스를 subscribe할 때까지 기다린 후에 요소를 emit할 수 있음

`ObservableType`에 구현된 애들
> 1. publish
> 2. replay
> 3. multicast

`ConnectableObservableType`에 구현 (ConnectableObservable에서만 사용 가능)
> -  refCount()

<br>

### 들어가기 전에) non-connectable 연산자 예시
- `interval` : 지정된 스케줄러에서 특정 period가 지난 후에 요소를 내보내는 옵저버블 시퀀스를 만드는 오퍼레이터
    - 특정 시간 간격을 두고 1씩 증가하는 정수 시퀀스를 방출하는 Observable을 생성
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
- 내부에서 subject를 만들어서 소스 옵저버블이 emit한 event를 subject에 전달하고, subject가 소스 옵저버블의 subscribers에게 event를 emit해줌 (multicast와 동일하나, parameter 필요 없이 내부에서 subject 만드는게 다름)
- 내부에서 생성되는 subject는 PublishSubject이다

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
- 내부에서 subject를 만들어서 소스 옵저버블이 emit한 event를 subject에 전달하고, subject가 소스 옵저버블의 subscribers에게 event를 emit해줌 (multicast와 동일하나, parameter 필요 없이 내부에서 subject 만드는게 다름)
- 내부에서 생성되는 subject각 ReplaySubject이다
  
![image](https://user-images.githubusercontent.com/46644241/113640113-b4b5ea80-96b5-11eb-9bd5-3d3f5741c461.png)

- If you apply the Replay operator to an Observable before you convert it into a connectable Observable, the resulting connectable Observable will always emit the same complete sequence to any future observers, even those observers that subscribe after the connectable Observable has begun to emit items to other subscribed observers.
- ~~....뭔소리여 후,,, 마음을 가다듬고 번역~~
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
- subject를 파라미터로 받음. 원본 Observerble에서 방출되는 이벤트가 구독자에게 바로 전달되는 것이 아니라 파라미터로 전달된 Subject에게 전달된다.
- 그리고 이 Subject가 등록된 다수의 구독자에게 이벤트를 전달한다. (broadcast)
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

## 추가1. RefCount
- connectable Observable을 일반 Observable처럼 동작하게 만드는 오퍼레이터
- connectable observable에서 작동하고 ordinary Observable를 반환함
- 즉, connect() 할 때 emit을 시작하는게 아니라, subscribe를 할 때 connect되어 emit 해주도록 하는 역할 (구독될 때 내부적으로 connect()를 호출하도록 설계됨)
- 첫 번째 observer가 observable을 구독하면 RefCount는 underlying connectable Observable에 연결됨
- 얼마나 많은 다른 observer가 subscribe하는지 추적하고, 마지막 observer가 끝나기 전까지 underlying connectable Observable를 disconnect하지 않음
- 구독자가 없으면 자동으로 이벤트 종료, 새로운 구독자 생기면 Sequence 다시 시작? => 검증 필요
- 주로 connect를 안쓰고도(명시하지 않고도) connectable observable의 이점(multicast)을 사용할 수 있도록 해주는 역할로 사용됨
    - 하지만 그런 역할로 쓰려면 subscribe하는 시점에 대해 잘 조절해서 정해야함
![image](https://user-images.githubusercontent.com/46644241/113874653-fb5c2f80-97f0-11eb-99bb-5bd991742c85.png)

## 추가2. autoCount (by Combine)
- parameter로 구독자 수 Int 값을 넣어주면 해당 수만큼의 구독자만 자동 connect해주고 종료되는 오퍼레이터
- refCount와 유사하게 움직임
- refCount와 차이점은 더이상 배출할 Observer가 없으면
    - `refCount`는 자동으로 자신을 해지를 하고 다시 새로운 Observer 이 오면 처음부터 자동으로 시작
    - `autoConnect`는 동작이 끝나면 dispose 가 되지만 다시 새로운게 오면 시작을 하진 않는다.

## 추가3. Share
- multicast() + refCount()
- 2개의 파라미터를 받음
    - `replay:` 기본값은 0임 0보다 큰 값이면 ReplaySubject를 생성하므로 replay()와 같고, 기본값이면 PublishSubject를 생성하므로 publish()와 같음
    - `scope:` 생명주기. 기본값은 .whileConnected임 .whileConnected를 쓰면 refCount()처럼 구독이 시작되면 connect 되었다가 구독이 끝나면 종료되는 반면, .forever는 multicast()처럼 하나의 subject를 공유함
