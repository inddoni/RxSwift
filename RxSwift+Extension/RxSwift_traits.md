# RxSwift Traits
0. Traits: Why, How they work
1. Single
2. Completable
3. Maybe


## 0. Traits: Why, How they work
### Why
- Swift는 애플리케이션의 정확성과 안정성 향상시키고, Rx를 보다 직관적이고 직접적인 경험으로 만드는데 사용할수 있는 강력한 유형 시스템 가지고 있다.
- Traits는 모든 context에서 사용할 수 있는 row 옵저버블과 비교할 때 context상의 의미, syntactic sugar 및 더 구체적인 use-cases를 제공할 뿐만 아니라 인터페이스 경계를 넘어 observable sequence의 속성들을 전달하고 보장하는데 도움을 준다.
    - 따라서, Traits는 전적으로 선택사항이다.
    - 모든 핵심 RxSwift/RxCocoa API들이 지원하므로, 프로그램의 모든 곳에서 Row 옵저버블을 자유롭게 사용할 수 있다.
- 참고) 이 문서에 설명된 일부 특성(ex. `Driver`)은 RxCocoa에만 해당되며, 일부 파트는 기본적인 일반 RxSwift이다. 그러나 필요한 경우, 다른 Rx implementation에서도 동일한 원칙을 쉽게 구현할 수 있다. There is no private API magic needed.
### How they work
- Traits are simply a wrapper struct - 간단한 wrapper 구조체이다.
- with a single read-only Observable sequence property. - 읽기 전용 옵저버블 시퀀스 속성을 랩핑한다.
```swift
struct Single<Element> {
    let source: Observable<Element>
}

struct Driver<Element> {
    let source: Observable<Element>
}
...
```
- 옵저버블 시퀀스에 대한 일종의 **builder 패턴 구현**으로 생각할 수 있다.
- Traits가 빌드될 때 `.asObservable()`을 호출하면 다시 원래의 옵저버블 시퀀스(vanilla observable sequence)로 변환된다.
## 1. Single
```swift
func getRepo(_ repo: String) -> Single<[String: Any]> {
    return Single<[String: Any]>.create { single in
        let task = URLSession.shared.dataTask(with: URL(string: "https://api.github.com/repos/\(repo)")!) { data, _, error in
            if let error = error {
                single(.error(error))
                return
            }

            guard let data = data,
                  let json = try? JSONSerialization.jsonObject(with: data, options: .mutableLeaves),
                  let result = json as? [String: Any] else {
                single(.error(DataError.cantParseJSON))
                return
            }

            single(.success(result))
        }

        task.resume()

        return Disposables.create { task.cancel() }
    }
}
```
```swift
getRepo("ReactiveX/RxSwift")
    .subscribe { event in
        switch event {
            case .success(let json):
                print("JSON: ", json)
            case .error(let error):
                print("Error: ", error)
        }
    }
    .disposed(by: disposeBag)
```
```swift
getRepo("ReactiveX/RxSwift")
    .subscribe(onSuccess: { json in
                   print("JSON: ", json)
               },
               onError: { error in
                   print("Error: ", error)
               })
    .disposed(by: disposeBag)
```
## 2. Completable
```swift
func cacheLocally() -> Completable {
    return Completable.create { completable in
       // Store some data locally
       ...
       ...

       guard success else {
           completable(.error(CacheError.failedCaching))
           return Disposables.create {}
       }

       completable(.completed)
       return Disposables.create {}
    }
}
```
```swift
cacheLocally()
    .subscribe { completable in
        switch completable {
            case .completed:
                print("Completed with no error")
            case .error(let error):
                print("Completed with an error: \(error.localizedDescription)")
        }
    }
    .disposed(by: disposeBag)
```
```swift
cacheLocally()
    .subscribe(onCompleted: {
                   print("Completed with no error")
               },
               onError: { error in
                   print("Completed with an error: \(error.localizedDescription)")
               })
    .disposed(by: disposeBag)
```
## 3. Maybe
```swift
func generateString() -> Maybe<String> {
    return Maybe<String>.create { maybe in
        maybe(.success("RxSwift"))

        // OR

        maybe(.completed)

        // OR

        maybe(.error(error))

        return Disposables.create {}
    }
}
```
```swift
generateString()
    .subscribe { maybe in
        switch maybe {
            case .success(let element):
                print("Completed with element \(element)")
            case .completed:
                print("Completed with no element")
            case .error(let error):
                print("Completed with an error \(error.localizedDescription)")
        }
    }
    .disposed(by: disposeBag)
```
```swift
generateString()
    .subscribe(onSuccess: { element in
                   print("Completed with element \(element)")
               },
               onError: { error in
                   print("Completed with an error \(error.localizedDescription)")
               },
               onCompleted: {
                   print("Completed with no element")
               })
    .disposed(by: disposeBag)
```


> Reference : [RxSwift Traits 문서](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Traits.md)