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
- 왜 이름이 reduce일까,,?
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
- 다음 시퀀스에서 element가 emit되기 전, 각 시퀀스가 성공적으로 종료될 때까지 대기하면서 inner Observable sequences를 순차적으로 결합
- 둘 이상의 옵저버블을 interleaving 하지 않고 emit 함
    - `interleaving` : 끼워 넣기 / 분산해서 처리하는 것 (비동기와 비슷한 느낌)
- 앞 옵저버블 시퀀스에서 언제 completed 되는지의 시점에 따라 뒤 옵저버블의 어떤 element들이 살아남을지 결정됨
- 두 옵저버블을 이어 붙이는 오퍼레이터
- first 옵저버블이 completed 된 이후에 second 옵저버블을 이어 붙임
  - first 옵저버블이 completed 되지 않은 상태에서 second 옵저버블에서 아이템이 방출되면 concat에서 유실될 수 있음
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
    
    subject2.onNext("I would be ignored") // 출력 안됨
    subject2.onNext("🐱")
    
    subject1.onCompleted()
    
    subject2.onNext("🐭")
}
```