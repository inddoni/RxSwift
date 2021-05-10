# RxCocoa Traits
1. [Driver](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#1-driver)
2. [Signal](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#2-signal)
3. [ControlProperty](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#3-controlproperty)
4. [ControlEvent](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#4-controlevent)

## 1. Driver

> Can't error out.  오류가 없다. 오류를 방출하지 않는다. <br>
> Observe occurs on **main scheduler**. observe가 메인 스케줄러에서 일어난다. <br>
> Shares side effects (`share(replay: 1, scope: .whileConnected)`). 사이드 이펙트를 공유함

- 가장 정교한 특성
- UI Layer에서 reactive code를 작성하는 직관적인 방법을 제공하거나
- 애플리케이션에서 데이터 스트림을 모델링하려는 모든 경우에 사용
  
**Why is it named Driver**
- 의도된 use case는 애플리케이션을 drive하는 시퀀스를 모델링하는 것
- E.g.
  - CoreData model에서 UI를 구동함
  - 다른 UI elements(바인딩)의 값을 이용해서 UI를 구동함
- 일반 운영체제 구동과 같이, 시퀀스 에러가 발생하면 어플리케이션은 사용자 입력에 응답하지 않는다.
- UI elements와 어플리케이션 로직은 일반적으로 thread로부터 안전하지 않기 때문에, 이런 element들이 main thread에서 observed 된다는 것 매우 중요함
- Driver는 사이드 이펙트를 공유하는 옵저버블 시퀀스를 builds함
  

**Practical usage example**
- This is a typical beginner example:
```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
    }

results
    .map { "\($0.count)" }
    .bind(to: resultCount.rx.text)
    .disposed(by: disposeBag)

results
    .bind(to: resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```
- 이 코드의 의도된 동작은 다음과 같음:
  - 사용자 입력을 제한한다.
  - 서버에 접속하여, 사용자 결과 리스트를 fetch해온다. (쿼리당 한 번)
  - 결과를 두 UI elements(결과 테이블 뷰, 결과의 수를 표시하는 레이블)에 바인딩한다.
- 그렇다면 이 코드의 문제점은?:
  - `fetchAutoCompleteItems` 옵저버블 시퀀스가 오류가 나면(connection failed or parsing error), 모든 바인딩이 해제되고 UI가 더 이상 새로운 쿼리에 응답하지 않는다.
  - `fetchAutoCompleteItems`가 일부 백그라운드 thread에서 결과를 리턴하면, 그 결과가 백그라운드 thread의 UI elements에 바인딩되어 의도치않은(비결정적) 충돌을 일으킬 수 있다.
  - 결과가 두 개의 UI element에 바인딩 된다. 즉, 각 사용자 쿼리에 대해 각 UI element에 하나씩 두 개의 HTTP request가 만들어진다.(의도치 않게)

<br>

- 더 적절한 버전의 코드는 다음과 같음:
```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .observeOn(MainScheduler.instance)  // results are returned on MainScheduler
            .catchErrorJustReturn([])           // in the worst case, errors are handled
    }
    .share(replay: 1)                           // HTTP requests are shared and results replayed
                                                // to all UI elements

results
    .map { "\($0.count)" }
    .bind(to: resultCount.rx.text)
    .disposed(by: disposeBag)

results
    .bind(to: resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```
- 이러한 모든 요구사항을 대규모 시스템에서 제대로 처리되는지 확인하는 것은 어려울 수 있다.
- 그러나, 컴파일러와 traits를 사용하여 이러한 요구사항이 충족되었는지 증명하는 더 간단한 방법이 있다.
- The following code looks almost the same:
```swift
let results = query.rx.text.asDriver()        // This converts a normal sequence into a `Driver` sequence.
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .asDriver(onErrorJustReturn: [])  // Builder just needs info about what to return in case of error.
    }

results
    .map { "\($0.count)" }
    .drive(resultCount.rx.text)               // If there is a `drive` method available instead of `bind(to:)`,
    .disposed(by: disposeBag)              // that means that the compiler has proven that all properties
                                              // are satisfied.
results
    .drive(resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```
- 그래서, 여기서 무슨 일이 일어나고 있는 걸까?
- 이 첫 번째 `asDriver` 메서드는 `ControlProperty` traits를 `Driver` traits로 변환합니다.
```swift
query.rx.text.asDriver()
```
- 얘네가 수행해야 할 특별한 사항은 없다.
- `Driver`는 `ControlProperty` trait의 모든 속성과 그 이상을 가지고 있습니다. 
- 기본적으로 옵저버블 시퀀스는 `Driver` trait으로 래핑되어 있다.
  
<br>

- 두 번째 변경사항은 다음과 같다:
```swift
.asDriver(onErrorJustReturn: [])
```
- 옵저버블 시퀀스는 아래 3가지 속성을 충족하는 한 `Driver` trait로 변환할 수 있다:
  - Can't error out.
  - Observe on main scheduler.
  - Sharing side effects (`share(replay: 1, scope: .whileConnected)`).
-  그럼 이 속성들이 충족되는지 어떻게 확인할까?
   -  일반 Rx 오퍼레이터를 사용하면 된다.
   -  `asDriver(onErrorJustReturn: [])`는 아래 코드와 동일하다.
```swift
let safeSequence = xs
  .observeOn(MainScheduler.instance)        // observe events on main scheduler
  .catchErrorJustReturn(onErrorJustReturn)  // can't error out
  .share(replay: 1, scope: .whileConnected) // side effects sharing

return Driver(raw: safeSequence)            // wrap it up
```
- 마지막 부분은 `bind(to:)`를 사용하는 대신 `drive`를 쓴 것이다.
- `drive`는 `Driver` trait에서만 정의가 된다.
- 즉, 코드의 어딘가에 `drive`가 있는 경우, 해당 옵저버블 시퀀스는 에러를 발생하지 않으며 UI 요소를 바인딩하기 안전한 main thread에서 observe한다.
- 그러나 이론적으로, 누군가가 ObservableType 또는 다른 인터페이스에서 `drive` 메소드를 정의할 수 있으므로 `let results: Driver<[Results]> = ...`로 일시적인 definition을 생성하는 것이 더 안전하다. (UI 요소에 바인딩하기 전에 완전한 증명을 위해 필요할 것..)
- 그러나 이것이 현실적인 시나리오인지 아닌지를 결정하는 것은 독자에게 맡기겠다.

## 2. Signal
- `Driver`와 비슷한데 한 가지만 다르다!
  - 구독 시 최신 이벤트를 replay하지 않지만 구독자는 여전히 시퀀스의 계산 리소스를 공유합니다.
- `Signal` 은:
  - Can't error out.
  - Delivers events on Main Scheduler.
  - Shares computational resources (share(scope: .whileConnected)).
  - Does NOT replay elements on subscription.

## 3. ControlProperty
- `UISearchBar + Rx`와 `UISegmentedControl + Rx`에서 아주 좋은 실용적인 예를 찾을 수 있습니다.
```swift
extension Reactive where Base: UISearchBar {
    /// Reactive wrapper for `text` property.
    public var value: ControlProperty<String?> {
        let source: Observable<String?> = Observable.deferred { [weak searchBar = self.base as UISearchBar] () -> Observable<String?> in
            let text = searchBar?.text
            
            return (searchBar?.rx.delegate.methodInvoked(#selector(UISearchBarDelegate.searchBar(_:textDidChange:))) ?? Observable.empty())
                    .map { a in
                        return a[1] as? String
                    }
                    .startWith(text)
        }

        let bindingObserver = Binder(self.base) { (searchBar, text: String?) in
            searchBar.text = text
        }
        
        return ControlProperty(values: source, valueSink: bindingObserver)
    }
}
```
```swift
extension Reactive where Base: UISegmentedControl {
    /// Reactive wrapper for `selectedSegmentIndex` property.
    public var selectedSegmentIndex: ControlProperty<Int> {
        value
    }
    
    /// Reactive wrapper for `selectedSegmentIndex` property.
    public var value: ControlProperty<Int> {
        return UIControl.rx.value(
            self.base,
            getter: { segmentedControl in
                segmentedControl.selectedSegmentIndex
            }, setter: { segmentedControl, value in
                segmentedControl.selectedSegmentIndex = value
            }
        )
    }
}
```
## 4. ControlEvent

**Practical usage example**
- 사용할 수있는 일반적인 사례는 다음과 같다
```swift
public extension Reactive where Base: UIViewController {
    
    /// Reactive wrapper for `viewDidLoad` message `UIViewController:viewDidLoad:`.
    public var viewDidLoad: ControlEvent<Void> {
        let source = self.methodInvoked(#selector(Base.viewDidLoad)).map { _ in }
        return ControlEvent(events: source)
    }
}
```
- `UICollectionView + Rx`에서 다음과 같이 찾을 수 있다.
```swift
extension Reactive where Base: UICollectionView {
    
    /// Reactive wrapper for `delegate` message `collectionView:didSelectItemAtIndexPath:`.
    public var itemSelected: ControlEvent<IndexPath> {
        let source = delegate.methodInvoked(#selector(UICollectionViewDelegate.collectionView(_:didSelectItemAt:)))
            .map { a in
                return a[1] as! IndexPath
            }
        
        return ControlEvent(events: source)
    }
```

<br>
<br>

> **Reference** : [RxSwift Traits 문서](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Traits.md) <br>
> **최종수정일** : 2021.04.20