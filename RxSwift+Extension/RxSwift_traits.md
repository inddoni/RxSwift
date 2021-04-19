# RxSwift Traits
0. [Traits: Why, How they work](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxSwift_traits.md#0-traits-why-how-they-work)
1. [Single](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxSwift_traits.md#1-single)
2. [Completable](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxSwift_traits.md#2-completable)
3. [Maybe](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxSwift_traits.md#3-maybe)


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
- 옵저버블 시퀀스에 대한 일종의 **builder 패턴 구현**이라고 생각할 수 있다.
- Traits가 빌드될 때 `.asObservable()` 을 호출하면 다시 원래의 옵저버블 시퀀스(vanilla observable sequence)로 변환된다.
## 1. Single
- 일련의 elements를 emitting하는 대신에 항상 **Single element**나 **Error**를 emit하도록 보장하는 옵저버블의 변형

>     Emits exactly one element, or an error. 정확히 하나의 요소 혹은 에러를 방출한다.
>     Doesn't share side effects. 사이드 이펙트를 공유하지 않는다.

- Common Use case
    - 하나의 응답이나 하나의 에러만 반환할 수 있는 HTTP Request를 수행할 때!
    - 단일 요소만 care하는 모든 경우를 모델링하는데 사용할 수 있음
    - 요소의 무한 스트림에는 사용할 수 없음

<br>

**Creating a Single**
- Single의 생성은 옵저버블 생성과 유사하다.
- 간단한 예:
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
- 그 후에 다음과 같이 사용할 수 있음:
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
- `subscribe(onSuccess:onError:)` 는 다음과 같이 사용할 수 있음:
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
- subscription은 **SingleEvent Enumeration**을 제공한다.
    - Single type의 요소를 포함하는 `.success` 또는 `.error`를 포함하고 있다.
    - 첫 번째 이벤트 이후에는 더 이상의 이벤트 emit이 발생하지 않는다.
- Row 옵저버블 시퀀스에서 `.asSingle()`을 사용하여 Single로 변환할 수 있다.

<br>

## 2. Completable
- **complete**이나 **error**만 emit할 수 있는 옵저버블의 변형이다.
- 어떤 element도 emit하지 않는 것이 보장된다.
>     Emits zero elements. 0개의 요소를 방출한다.
>     Emits a completion event, or an error. 완료 이벤트나 에러만 방출한다.
>     Doesn't share side effects. 사이드 이펙트를 공유하지 않는다.
- useful use case
    - 작업이 완료된 사실만 신경 쓰고, 해당 완료의 결과로 나온 elements는 신경 쓰지 않아도 되는 경우를 모델링할 때!
    - 요소를 방출할 수 없는 `Observable <Void>`을 사용하는 것과 비교할 수 있다.

<br>

**Creating a Completable**
- Completable의 생성은 옵저버블 생성과 유사하다.
- 간단한 예:

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
- 그 후에 다음과 같이 사용할 수 있음:
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
- `subscribe(onCompleted:onError:)` 는 다음과 같이 사용할 수 있음:
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
- subscription은 **CompletableEvent Enumeration**을 제공한다.
    - 오류 없이 작업이 완료되었음을 나타내는 `.completed` 또는 `.error`를 포함하고 있다.
    - 첫 번째 이벤트 이후에는 더 이상의 이벤트 emit이 발생하지 않는다.

## 3. Maybe
- Maybe는 Single과 Completable 사이에있는 옵저버블의 변형입니다. 
- single element를 emit하거나, 요소를 방출하지 않고 complete하거나, error를 emit 할 수 있습니다.
    - 이 세 가지 중에 하나는 Maybe를 **terminate** 시킴
    - 즉, completed한 Maybe도 요소를 방출할 수 없고, 요소를 emitted한 Maybe도 요소를 내 보낸 Maybe도 Completion event를 내보낼 수 없다.
>     Emits either a completed event, a single element or an error. 완료 이벤트, 단일 요소 또는 에러만 방출한다.
>     Doesn't share side effects. 사이드 이펙트를 공유하지 않는다.
- Maybe를 사용하여 요소를 방출할 수 있는 작업을 모델링 할 수 있지만,
- **반드시 요소를 방출 할 필요는 없습니다.**
<br>

**Creating a Maybe**
- Maybe의 생성은 옵저버블 생성과 유사하다.
- 간단한 예:
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
- 그 후에 다음과 같이 사용할 수 있음:
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
- `subscribe(onSuccess:onError:onCompleted:)` 는 다음과 같이 사용할 수 있음:
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
- Row 옵저버블 시퀀스에서 `.asMaybe()`을 사용하여 Maybe로 변환할 수 있다.

<br>
<br>

> **Reference** : [RxSwift Traits 문서](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Traits.md) <br>
> **최종수정일** : 2021.04.20