## android_flux_example

# ListActivity

ViewModel は[Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html)を利用して生成。

```kotlin
viewModel = ViewModelProviders.of(this).get(ListViewModel::class.java)
```

RecyclerView の ScrollListener を設定して、スクロールしたら ViewModel の onScrollToLast メソッドを呼び出すようにしている。

```kotlin
listView.let {
    val layoutManager = LinearLayoutManager(this)
    it.layoutManager = layoutManager
    it.adapter = adapter
    layoutManager.stackFromEnd = false
    it.addOnScrollListener(EndlessScrollListener(2, layoutManager) {
        viewModel.onScrollToLast()
    })
}
```

ViewModel の listItems を observe し、これが更新されたら RecyclerView に反映されるようにしている。

```kotlin
viewModel.listItems.observe(this, Observer {
    it ?: return@Observer
    adapter.items = it
})
```

# ListViewModel

ViewModel の listItems は[Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html)の LiveData を利用。※unsubscribe が不要

```kotlin
val listItems: MutableLiveData<List<ListItemType>> = MutableLiveData()
```

RecyclerView のスクロールにより ListActivity から呼び出される onScrollToLast メソッドは ActionCreator に処理を委譲している。

```kotlin
fun onScrollToLast() {
    store.actionCreator.onScrollToLast()
}
```

# ListActionCreator

ListActionCreator は、Store の actionCreator 委譲プロパティ内で生成される。（後述する Reducer や Getter も同様）

```kotlin
abstract class Store<AT, out AC, R, out G> where AC : DisposableMapper, R : DisposableMapper, G : DisposableMapper {
    private val dispatcher: PublishSubject<AT> = PublishSubject.create()
    private val reducer: R by lazy {
        createReducer(dispatcher)
    }
    val actionCreator: AC by lazy {
        createActionCreator({
            dispatcher.onNext(it)
        }, reducer)
    }
    val getter: G by lazy {
        createGetter(reducer)
    }
// omit
```

createActionCreator の実装は以下の通り。

```kotlin
class ListStore(private val repository: Repository) : Store<ListActionType, ListActionCreator, ListReducer, ListGetter>() {
    override fun createActionCreator(dispatch: (ListActionType) -> Unit, reducer: ListReducer): ListActionCreator = ListActionCreator(dispatch, reducer, repository)
// omit
```

ViewModel から呼び出されている onScrollToLast メソッドは以下の通り。

```kotlin
fun onScrollToLast() {
    if (reducer.isNextLoading.value) return
    dispatch(ListActionType.StartNextLoad())
    repository.list(reducer.page + 1)
            .subscribe({
                dispatch(ListActionType.SuccessNextLoad(it))
            }, {
                dispatch(ListActionType.Error(it))
            }).let { disposables.add(it) }
}
```

ここで dispatch 関数は、

```kotlin
{
    dispatcher.onNext(it)
}
```

になるので、dispatcher（PublishSubject）を subscriber（後述する Reducer）に ListActionType が渡されることになる。  
※Subject は「Subscriber と Observable の 2 つの機能を併せ持ったもの」

なお、ListActionType はシールドクラスとして定義されている。

```kotlin
sealed class ListActionType {
    class StartInitialLoad : ListActionType()
    class SuccessInitialLoad(val elements: List<Element>) : ListActionType()
    class StartNextLoad : ListActionType()
    class SuccessNextLoad(val elements: List<Element>) : ListActionType()
    class Error(val error: Throwable) : ListActionType()
}
```

また、以下の部分がデータ取得に相当している。

```kotlin
repository.list(reducer.page + 1)
```

list メソッドは以下の様になっている。

```kotlin
fun list(page: Int): Single<List<Element>> =
        Single.just(Array(LIMIT) { Element(LIMIT * (page - 1) + it) }.toList()).delay(1, TimeUnit.SECONDS)
```

ActionCreator は、list メソッド  の結果を subscribe して、結果取得ができたら ListActionType.SuccessNextLoad を dispatch する。

# ListReducer

前述の通り Store により生成される。  
生成の際に PublishSubject のインスタンスが渡されるがこれを subscribe している。

以下、ListActionType.SuccessNextLoad を subscribe している部分。

```kotlin
action.ofType(ListActionType.SuccessNextLoad::class.java)
        .subscribe {
            mIsNextLoading.value = false
            mElements.value += it.elements
            page++
        }.let { disposables.add(it) }
```

mElements に Repository から得られた Element の List を  追加しています。

```kotlin
private val mElements: Variable<List<Element>> = Variable(listOf())
val elements: ImmutableVariable<List<Element>>
    get() = mElements
```

Variable は変更可能、ImmutableVariable がその名の通り参照 ONLY になっていて、Reducer 外部には ImmutableVariable を公開する。  
※distinctUntilChanged メソッドをかますことで値の変更があったときのみ通知されるようにしている

```kotlin
interface ImmutableVariable<T> {
    val value: T
    val observable: Observable<T>
}

class Variable<T>(initialValue: T) : ImmutableVariable<T> {
    private val subject = BehaviorSubject.createDefault(initialValue)
    override var value: T
        get() = subject.value
        set(value) {
            subject.toSerialized().onNext(value)
        }

    override val observable: Observable<T>
        get() = subject.distinctUntilChanged()
}
```

# ListGetter

Reducer の State を監視して、変更があった場合に View に反映する役目を担う。

```kotlin
class ListGetter(reducer: ListReducer) : DisposableMapper() {

    val listItems: Observable<List<ListItemType>> =
            Observable.combineLatest(
                    reducer.elements.observable,
                    reducer.isNextLoading.observable,
                    BiFunction { elements, isNextLoading ->
                        var listItems: List<ListItemType> = elements.map { ListItemType.Element(it) }
                        if (isNextLoading) {
                            listItems += ListItemType.Loading()
                        }

                        listItems
                    })
// omit
```

isNextLoading が true ならリストの最後に Loading を付け足している。

# ListViewModel（再）

ViewModel は Getter の listItems を subscribe している。

```kotlin
store.getter.listItems
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe({
            listItems.value = it
        }, {
        }).let { disposables.add(it) }
```
