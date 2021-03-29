# 07. Mathematical and Aggregate Operators
> Observable이 내보낸 항목의 전체 시퀀스에서 작동하는 연산자

1. toArray
2. reduce
3. concat
<br>
<br>

## 1. toArray
- 옵저버블 시퀀스를 배열로 변환
- 변환한 배열을 새로운 하나의 element로 갖도록 옵저버블 시퀀스로 emit 후 completed
- `To` Operator
    - 다양한 object(struct 등), data structure로 변환하는데 사용하는 연산자
    - 내용 보충 필요 

```swift
example("toArray") {
    let disposeBag = DisposeBag()
    
    Observable.range(start: 1, count: 10)
        .toArray()
        .subscribe { print($0) }
        .disposed(by: disposeBag)
}
```
## 2. reduce
- seed 초기 값으로 시작한 다음, 옵저버블 시퀀스가 보내는 모든 elements에 accumulator 클로저 적용
- 집계 결과를 single element 옵저버블 시퀀스로 반환
- complete 될 때까지 방출되는 element에 accumulator 적용하는 과정을 반복
```swift
example("reduce") {
    let disposeBag = DisposeBag()
    
    Observable.of(10, 100, 1000)
        .reduce(1, accumulator: +)
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
}
```
## 3. concat
```swift
example("concat") {
    let disposeBag = DisposeBag()
    
    let subject1 = BehaviorSubject(value: "🍎")
    let subject2 = BehaviorSubject(value: "🐶")
    
    let subjectsSubject = BehaviorSubject(value: subject1)
    
    subjectsSubject.asObservable()
        .concat()
        .subscribe { print($0) }
        .disposed(by: disposeBag)
    
    subject1.onNext("🍐")
    subject1.onNext("🍊")
    
    subjectsSubject.onNext(subject2)
    
    subject2.onNext("I would be ignored")
    subject2.onNext("🐱")
    
    subject1.onCompleted()
    
    subject2.onNext("🐭")
}
```