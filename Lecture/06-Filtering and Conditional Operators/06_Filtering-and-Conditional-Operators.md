# 06. Filtering and Conditional Operators
> 소스 옵저버블 시퀀스에서 elements를 선택적으로 내보내는 오퍼레이터

1. filter
2. distinctUntilChanged
3. elementAt
4. single
5. take
6. takeLast
7. takeWhile
8. takeUntil
9. skip
10. skipWhile
11. skipWhileWithIndex
12. skipUntil

## 1. filter
- 지정한 조건을 만족하는 element만 내보냄
- predicate function에서 true / false 반환
    - `predicate` : 조건에 대해 각 요소를 테스트하는 function
```swift
public func filter(_ predicate: @escaping (Self.Element) throws -> Bool) -> 
    RxSwift.Observable<Self.Element>
```
```swift
example("filter") {
    let disposeBag = DisposeBag()
    
    Observable.of(
        "🐱", "🐰", "🐶",
        "🐸", "🐱", "🐰",
        "🐹", "🐸", "🐱")
        .filter {
            $0 == "🐱"
        }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```

## 2. distinctUntilChanged
- 순차적으로 중복 요소가 emit되는 것을 방지하기 위한 오퍼레이터
- 현재 방출된 이벤트와 이전 이벤트와 비교해서 동일하면 필터링
```swift
example("distinctUntilChanged") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐷", "🐱", "🐱", "🐱", "🐵", "🐱")
        //.distinctUntilChanged()
        .distinctUntilChanged({ (previous, current) -> Bool in
            origin == current // false면 흐름
        })
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 3. elementAt
- 지정한 index의 요소만 내보내는 오퍼레이터
- 지정한 요소 하나를 내보내고 바로 completed 됨
```swift
example("elementAt") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .elementAt(3)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 4. single
- 옵저버블에서 내보낸 첫 번째 요소 혹은 조건에 만족하는 첫 번째 요소만 내보내는 오퍼레이터
- 정확히 하나의 element만 내보내지 않을 때 Error 발생
    - 조건 없이 single()만 썼을 때, 옵저버블의 element가 하나가 아니면 에러 `Sequence contains more than one element`
    - 조건에 만족하는 요소가 하나 이상일 경우 `Sequence contains more than one element.`
    - 조건에 만족하는 요소가 하나도 없을 경우 `Sequence doesn't contain any elements.`
- element를 emit하고 그 후에 Error가 발생함
```swift
example("single") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
```swift
example("single with conditions") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single { $0 == "🐸" }
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
    Observable.of("🐱", "🐰", "🐶", "🐱", "🐰", "🐶")
        .single { $0 == "🐰" }
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .single { $0 == "🔵" }
        .subscribe { print($0) }
        .disposed(by: disposeBag)
}
```
## 5. take
- 시퀀스의 시작 부분에서 지정된 수의 element만 내보내는 오퍼레이터
- 처음부터 지정된 수(n개)의 item을 emit 후 completed 됨
    - 남은 애들은 ignore,,,, 하면서 옵저버블(자기자신)을 수정하는 개념
- 옵저버블이 가지고 있는 element보다 큰 수를 입력해도 error 발생안함
- 반대개념 `skip(n)` : n개의 element는 무시하고 그 이후 부터 내보내는 오퍼레이터
```swift
example("take") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .take(3)
        .subscribe(onNext: { print($0) }, onCompleted: { print("completed")})
        .disposed(by: disposeBag)
}
```
## 6. takeLast
- 끝에서부터 지정된 수의 element만 내보내는 오퍼레이터
- 끝부터 지정된 수(n개)의 item을 emit 후 completed 됨
- 앞에 있는 element들을 ignore 하고 싶을 때 사용
```swift
example("takeLast") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .takeLast(3)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 7. takeWhile
- 지정된 조건이 `true`인 동안 옵저버블의 처음부터 elements를 내보내는 오퍼레이터
- `false`가 나오는 동시에 completed 됨 (즉, false가 될 때까지 elements를 emits)

```swift
example("takeWhile") {
    let disposeBag = DisposeBag()
    
    Observable.of(1, 2, 3, 4, 5, 6)
        .takeWhile { $0 < 4 }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 8. takeUntil
- reference 옵저버블이 요소를 emit할 때까지 source 옵저버블 시퀀스에서 요소를 emits
- takeUntil은 source observable을 subscribe하고 미러링을 함과 동시에 reference observable을 모니터링함
- 그러다가 reference observable가 item을 내보내거나 completed 되면, source observable의 미러링을 중지하고 completed 된 옵저버블을 return 함
- reference 옵저버블(second 옵저버블)이 emit하거나 completed 되면, 그 옵저버블이 emit한 item은 버림 (레퍼런스 옵저버블의 역할은 특별한 게 아니라, 이벤트 발생하는지 확인용,,?)

```swift
example("takeUntil") {
    let disposeBag = DisposeBag()
    
    let sourceSequence = PublishSubject<String>()
    let referenceSequence = PublishSubject<String>()
    
    sourceSequence
        .takeUntil(referenceSequence)
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
    sourceSequence.onNext("🐱")
    sourceSequence.onNext("🐰")
    sourceSequence.onNext("🐶")
    
    referenceSequence.onNext("🔴")
    
    sourceSequence.onNext("🐸")
    sourceSequence.onNext("🐷")
    sourceSequence.onNext("🐵")
}
```
## 9. skip
- 옵저버블의 시작 부분에서 지정된 수의 element의 emit을 막음
- 처음부터 지정된 수(n개)의 item을 ignore 후 emit & completed 하는 옵저버블로 자기자신을 수정하는 개념
```swift
example("skip") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .skip(2)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 10. skipWhile
- 시퀀스의 시작부분부터 적용
- 지정된 조건을 충족하면 emit을 ignore (충족하는 elements를 버림)
- 충족하지 않는 elements가 들어오면, 그 때부터 소스 옵저버블을 미러링함(emit)
```swift
example("skipWhile") {
    let disposeBag = DisposeBag()

    Observable.of(1, 2, 3, 4, 5, 6)
        .skipWhile { $0 < 4 }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 11. skipWhileWithIndex
- 
```swift
example("skipWhileWithIndex") {
    let disposeBag = DisposeBag()
    
    Observable.of("🐱", "🐰", "🐶", "🐸", "🐷", "🐵")
        .enumerated()
        .skipWhile { $0.index < 3 }
        .map { $0.element }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 12. skipUntil
- 
```swift
example("skipUntil") {
    let disposeBag = DisposeBag()
    
    let sourceSequence = PublishSubject<String>()
    let referenceSequence = PublishSubject<String>()
    
    sourceSequence
        .skipUntil(referenceSequence)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
    
    sourceSequence.onNext("🐱")
    sourceSequence.onNext("🐰")
    sourceSequence.onNext("🐶")
    
    referenceSequence.onNext("🔴")
    
    sourceSequence.onNext("🐸")
    sourceSequence.onNext("🐷")
    sourceSequence.onNext("🐵")
}
```