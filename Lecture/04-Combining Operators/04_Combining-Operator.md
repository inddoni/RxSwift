# 03. Combination Operators
> multiple source Observables을 Single Observable로 결합하는 오퍼레이터

1. startWith
2. merge
3. zip
4. combineLatest`
5. switchLatest
6. withLatestFrom


## 1. startWith
- source 옵저버블에서 elements를 방출하기 전에, 특정한 elements를 방출하는 것
- 예) startWith(1) => 1을 먼저 emits 후 source 옵저버블의 elements들을 emit함
- **last-in-first-out** : 여러 개의 startWith을 쓰면, 가장 마지막에 쓴 startWith()의 elements부터 emit
- 반대로, 옵저버블의 모든 elements를 emit하고나서 특정한 elements를 방출하려면 `concat` 오버레이터를 사용

```swift
example("startWith") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐶", "🐱", "🐭", "🐹")
        .startWith("1️⃣")
        .startWith("2️⃣")
        .startWith("3️⃣", "🅰️", "🅱️")
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
<br>

## 2. merge
- 기존의 source 시퀀스를 하나의 새로운 시퀀스로 결합하는 것
- 여러 옵저버블을 하나의 옵저버블로
- 각 옵저버블에서 emit 되는 대로 merging
- merge되는 중에 어떤 source 옵저버블에서 onError notification이 발생하면 바아로 observer에게 전달되고 병합된 옵저버블은 종료됨
    - `MergeDelayError` : onError notification을 예약처리하고, 모든 옵저버블의 merge가 완료될 때까지 기다리는 것. 다 끝나고 나서 에러 발생시키고 종료됨
```swift
example("merge") {
    let disposeBag = DisposeBag()
    
    let subject1 = PublishSubject<String>()
    let subject2 = PublishSubject<String>()
    
    Observable.of(subject1, subject2)
        .merge()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    subject1.onNext("🅰️")
    subject1.onNext("🅱️")
    subject2.onNext("①")
    subject2.onNext("②")
    subject1.onNext("🆎")
    subject2.onNext("③")
}
```

## 3. zip
- 최대 8개의 source 옵저버블 시퀀스를 하나의 새로운 옵저버블 시퀀스로 결합할 수 있음
- 여러 옵저버블의 인덱스 값이 같은 elements를 결합 (strict sequence)
- 새로운 옵저버블의 최대 elements 수 == 가장 적은 elements를 가진 source 옵저버블에서 emits한 수

```swift
example("zip") {
    let disposeBag = DisposeBag()
    
    let stringSubject = PublishSubject<String>()
    let intSubject = PublishSubject<Int>()
    
    Observable.zip(stringSubject, intSubject) { stringElement, intElement in
        "\(stringElement) \(intElement)"
        }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    stringSubject.onNext("🅰️")
    stringSubject.onNext("🅱️")
    intSubject.onNext(1)
    intSubject.onNext(2)
    stringSubject.onNext("🆎")
    intSubject.onNext(3)
}
```
<br>

## 4. combineLatest
- 최대 8개의 source 옵저버블 시퀀스를 하나의 새로운 옵저버블 시퀀스로 결합할 수 있음
- 모든 source 시퀀스가 적어도 하나의 element를 emit하면 지정된 함수를 통해 결합 시작
- 결합할 때, 결합된 옵저버블 시퀀스에서 각 source 옵저버블 시퀀스의 Latest element를 방출해서 최근 애들끼리 결합


```swift
example("combineLatest") {
    let disposeBag = DisposeBag()
    
    let stringSubject = PublishSubject<String>()
    let intSubject = PublishSubject<Int>()
    
    Observable.combineLatest(stringSubject, intSubject) { stringElement, intElement in
            "\(stringElement) \(intElement)"
        }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    stringSubject.onNext("🅰️")
    stringSubject.onNext("🅱️")
    intSubject.onNext(1)
    intSubject.onNext(2)
    stringSubject.onNext("🆎")
}
```
<br>

- **combineLatest의 변형**
    - Array 또는 옵저버블 시퀀스의 다른 collection을 사용할 때
    - 컬렉션을 사용하는 combineLatest의 값은 selector function을 사용하기 때문에 모든 source 옵저버블이 같은 type이어야 함
```swift

    let disposeBag = DisposeBag()
    
    let stringObservable = Observable.just("❤️")
    let fruitObservable = Observable.from(["🍎", "🍐", "🍊"])
    let animalObservable = Observable.of("🐶", "🐱", "🐭", "🐹")
    
    Observable.combineLatest([stringObservable, fruitObservable, animalObservable]) {
            "\($0[0]) \($0[1]) \($0[2])"
        }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```

## 5. switchLatest
```swift
example("switchLatest") {
    let disposeBag = DisposeBag()
    
    let subject1 = BehaviorSubject(value: "⚽️")
    let subject2 = BehaviorSubject(value: "🍎")
    
    let subjectsSubject = BehaviorSubject(value: subject1)
        
    subjectsSubject.asObservable()
        .switchLatest()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    subject1.onNext("🏈")
    subject1.onNext("🏀")
    subjectsSubject.onNext(subject2)
    subject1.onNext("⚾️")
    subject2.onNext("🍐")
}
```

## 6. withLatestFrom
```swift
example("withLatestFrom") {
    let disposeBag = DisposeBag()
    
    let foodSubject = PublishSubject<String>()
    let drinksSubject = PublishSubject<String>()
    
    foodSubject.asObservable()
        .withLatestFrom(drinksSubject) { "\($0) + \($1)" }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    foodSubject.onNext("🥗")
    drinksSubject.onNext("☕️")
    foodSubject.onNext("🥐")
    drinksSubject.onNext("🍷")
    foodSubject.onNext("🍔")
    foodSubject.onNext("🍟")
    drinksSubject.onNext("🍾")
}
```