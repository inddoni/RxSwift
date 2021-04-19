# RxCocoa Traits
1. [Driver](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#1-driver)
2. [Signal](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#2-signal)
3. [ControlProperty](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#3-controlproperty)
4. [ControlEvent](https://github.com/inddoni/RxSwift/blob/main/RxSwift%2BExtension/RxCocoa_traits.md#4-controlevent)

## 1. Driver
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