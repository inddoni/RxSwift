# 05. Trainforming Operator
> 옵저버블 시퀀스에 의해 생성된 Next 이벤트 elements를 변환하는 오퍼레이터

1. map
2. flatMap and flatMapLatest
3. scan

## 1. map
- 옵저버블 시퀀스에 의해 emit된 element에 closure 적용하여 transforming
  - 각 elements에 closure를 적용
- transformed elements를 가진 새로운 **옵저버블** 시퀀스를 return함 

```swift
example("map") {
    let disposeBag = DisposeBag()
    Observable.of(1, 2, 3)
        .map { $0 * $0 }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```

## 2. flatMap and flatMapLatest
- 기존 옵저버블의 elements를 Observabla sequence로 변환하고 두 옵저버블의 emits을 single 옵저버블로 병합 (옵저버블에서 발행한 아이템을 다른 옵저버블의 아이템으로 만들고, 만들어진 옵저버블에서 아이템을 발행함)
- 옵저버블 시퀀스를 emit하는 옵저버블이 있고, 다른 옵저버블 시퀀스의 new emissions에 react하도록 할 때 유용하게 사용
<img width="640" alt="flatMap" src="https://user-images.githubusercontent.com/46644241/112847485-1688ba00-90e2-11eb-91de-a8cd8c4c07b4.png">
- `flatMapLatest`는 가장 최근의 내부 Observable 시퀀스에서만 요소를 방출한다는 것이 다름
    - flatmap에서 가장 최근의 값만을 확인하고 싶을 때사용
- `flatMapLatest` = `map` + `switchLatest`
    - `switchLatest`는 가장 최근의 옵저버블에서 값을 생성하고 이전의 옵저버블은 구독 해제함
```swift
example("flatMap and flatMapLatest") {
    let disposeBag = DisposeBag()
    
    struct Player {
        init(score: Int) {
            self.score = BehaviorSubject(value: score)
        }

        let score: BehaviorSubject<Int>
    }
    
    let 👦🏻 = Player(score: 80)
    let 👧🏼 = Player(score: 90)
    let player = BehaviorSubject(value: 👦🏻)
    
    player.asObservable()
        .flatMap { $0.score.asObservable() } // Change flatMap to flatMapLatest and observe change in printed output
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    👦🏻.score.onNext(85)
    player.onNext(👧🏼)
    👦🏻.score.onNext(95) // Will be printed when using flatMap, but will not be printed when using flatMapLatest
    👧🏼.score.onNext(100)
}
```

## 3. scan
- 초기 seed 값으로 시작, 옵저버블에서 내보낸 element에 누적 계산 closure 적용
- 각 중간 결과를 single element 옵저버블로 반환함

```swift
example("scan") {
    let disposeBag = DisposeBag()
    
    Observable.of(10, 100, 1000)
        .scan(1) { aggregateValue, newValue in
            aggregateValue + newValue
        }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```

## 4. compactMap()
- 옵셔널이고, 이 중 nil이 아닌 것들만 array로 반환
- transform 클로저의 반환 값이 optional이야 하고, 이중에서 변환 결과가 nil일 경우 최종 결과에 포함되지 않는다는 점이 핵심