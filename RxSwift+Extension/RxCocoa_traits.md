# RxCocoa Traits
1. [Driver](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#1-driver)
2. [Signal](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#2-signal)
3. [ControlProperty](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#3-controlproperty)
4. [ControlEvent](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#4-controlevent)

## 1. Driver

> Can't error out. <br> 오류가 없다. 오류를 방출하지 않는다.
> Observe occurs on **main scheduler**. <br> observe가 메인 스케줄러에서 일어난다.
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
- UI elements와 어플리케이션 로직은 일반적으로 thread로부터 안전하지 않기 때문에 이러한 element들이 main thread에서 observed되는 것 또한 매우 중요함
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
  - `fetchAutoCompleteItems` 옵저버블 시퀀스가 오류가 나면(connection failed or parsing error), 모든 바인딩 해제하고 UI가 더 이상 새 쿼리에 응답하지 않는다.
  - `fetchAutoCompleteItems`가 일부 백그라운드 thread에서 결과를 리턴하면, 그 결과가 백그라운드 thread에 UI elements가 바인딩되어 의도치않은 충돌이 일어날 수 있다.
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
- 여기서 무슨 일이 일어나고 있나?
- 이 첫 번째 `asDriver` 메서드는 `ControlProperty` traits를 `Driver` traits로 변환합니다.
```swift
query.rx.text.asDriver()
```
- 두 번째 변경사항은 다음과 같다:
```swift
.asDriver(onErrorJustReturn: [])
```
```swift
let safeSequence = xs
  .observeOn(MainScheduler.instance)        // observe events on main scheduler
  .catchErrorJustReturn(onErrorJustReturn)  // can't error out
  .share(replay: 1, scope: .whileConnected) // side effects sharing

return Driver(raw: safeSequence)            // wrap it up
```
## 2. Signal

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