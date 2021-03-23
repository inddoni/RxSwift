# 05. Trainforming Operator
> 옵저버블 시퀀스에 의해 생성된 Next 이벤트 elements를 변환하는 오퍼레이터

1. map
2. flatMap and flatMapLatest
3. scan

## 1. map
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