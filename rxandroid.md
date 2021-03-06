# RxAndroid

https://github.com/ReactiveX/RxAndroid

生命週期的連動。

## AppObservable

```java
AppObservable.bindActivity()
AppObservable.bindFragment()
```

主要檢查 `Fragment.isAdded()`, `Activity.isFinishing()`。

*註：筆者不是很清楚，為什麼不用 overloading: `AppObservable.bind(Activity/Frgment/v4.Fragment)` 來取代 `AppObservable.bindFragment(Fragment)`,
`AppObservable.bindFragment(v4.Fragment)`,
`AppObservable.bindActivity(Activity)`*

## LifecycleObservable

```java
LifecycleObservable.bindActivityLifecycle()
LifecycleObservable.bindFragmentLifecycle()
```

當 Activty/Fragment 對應的生命週期結束時，自動 `unsubscribe()`。

`LifecycleObservable` 哪時候訂閱哪時候取消對照表：

```java
CREATE -> LifecycleEvent.DESTROY;
START -> LifecycleEvent.STOP;
RESUME -> LifecycleEvent.PAUSE;
PAUSE -> LifecycleEvent.STOP;
STOP -> LifecycleEvent.DESTROY;
```

手動自己 `unsubscribe()`， 如果 Activity 要結束，把一些 subscriptions 取消：

```java
class SimpleActivity extends Activity {
    CompsotionSubscription mSubscriptions = new CompositeSubscription();

    @Override
    protected void onResume() {
        super.onResume();

        bind(Observable.just("Hello, world"), s -> textView.setText(s));
    }

    protected <T> void bind(Observable<T> obs, Action1<T> onNext) {
        mCompositeSubscription.add(AppObservable.bindActivity(this, obs).subscribe(onNext));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        mCompositeSubscription.unsubscribe();
    }
}
```

`LifecycleObservable` + `RxActivity`:

```java
class SimpleActivity extends RxActivity {

    @Override
    public void onResume() {
        LifecycleObservable.bindActivityLifecycle(lifecycle(),
            AppObservable.bindActivity(this, Observable.just("Hello, world")))
        ).subscribe(s -> textView.setText(s));
    }
}
```

## ViewObservable, WidgetObservable

View 的連動. 當 View 顯示時 `subscribe()` 離開時 `unsubscribe()`

```java
ViewObservable.bindView()
```

Event 的連動.

```java
ViewObservable.clicks()
```

## ogaclejapan/RxBinding

https://github.com/ogaclejapan/RxBinding

類似於 ViewObservable。主要以 MVVM 雙向連動作努力。

Before：

```java
class HogeActivity extends Activity {

    @InjectView(R.id.text)
    private TextView mTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.inject(this);

        AppObservable.bindActivity(this, Observable.just("hoge"))
            .subscribeOn(Schedulers.io())
            .subscribe(setTextAction());
    }

    Action1<String> setTextAction() {
        return text -> mTextView.setText(text);
    }
}
```

After:

```java
class HogeActivity extends Activity {
    // ...

    private Rx<TextView> mRxTextView

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.inject(this);

        mRxTextView = RxView.of(mTextView);
        Subscription s = mRxTextView.bind(Observable.just("hoge"), RxActions.setText());
    }
}
```

## RxLifecycle

## See Also

* [RxActivity](https://github.com/ReactiveX/RxAndroid/blob/master/rxandroid-framework/src/main/java/rx/android/app/RxActivity.java)
* [RxFragmentActivity](https://github.com/ReactiveX/RxAndroid/blob/master/rxandroid-framework/src/main/java/rx/android/app/support/RxFragmentActivity.java)
* [RxFragment](https://github.com/ReactiveX/RxAndroid/blob/master/rxandroid-framework/src/main/java/rx/android/app/support/RxFragment.java)
* [ogaclejapan/RxBinding](https://github.com/ogaclejapan/RxBinding)
* [JakeWharton/RxBinding](https://github.com/JakeWharton/RxBinding)
* https://github.com/trello/RxLifecycle
* https://github.com/ReactiveX/RxAndroid/issues/172

*註：並沒有 RxAppCompatActivity*
