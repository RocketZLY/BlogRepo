# rxjava操作符结合使用场景简介

#### 前言

> 本文将通过实际的例子来介绍rx相关的操作符,如果对rxjava还不熟悉的同学请先查看rxjava相关基础姿势再来查看本文

#### 准备

> 本文依赖rxjava版本如下

```groovy
implementation 'io.reactivex.rxjava2:rxjava:2.1.15'
implementation 'io.reactivex.rxjava2:rxandroid:2.0.2'
```

#### 目录

> 目前根据实际使用场景一共总结了7个例子,后面如果还有的话我会持续更新

1. 网络请求嵌套(A请求成功后再进行B请求的情况)
2. 网络请求合并(A请求和B请求成功后合并数据在更新ui)
3. 三级缓存,从内存/磁盘/网络中读取数据
4. 组合判断(当A输入框和B输入框输入内容满足条件的时候Button才变成可点击的)
5. 网络请求出错重试
6. 联想搜索优化(用户x秒后没有再输入新的内容,进行搜索请求)
7. 防止多次点击

#### 1. 网络请求嵌套

> 先执行请求1成功后执行请求2.
>
> 这里通过`flatMap`操作符实现,将一个`Observable`转换成另一个`Observable`

网络请求的话这里自己模拟的代码如下

```kotlin
object ExampleRepo {

    fun getRequest1() = Observable.just(1)

    fun getRequest2(i: Int) = Observable.just(2)

}
```

执行请求代码如下

```kotlin
ExampleRepo.getRequest1()
.flatMap {
    ExampleRepo.getRequest2(it)
}
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(
    {
        Log.i("zhuliyuan", "onNext: $it")
    },
    {
        Log.i("zhuliyuan", "onError: $it")
    },
    {
        Log.i("zhuliyuan", "onComplete")
    },
    {
        Log.i("zhuliyuan", "onSubscribe")
    }
)
```

结果与预期并无差异

![](http://of1ktyksz.bkt.clouddn.com/rxjava_flatMapResult.png)

#### 2. 网络请求合并

> 将请求1和请求2成功后的数据合并供ui显示.
>
> 这里会用到`zip`操作符,合并多个被观察者`Observable`发送的事件,并最终发送

模拟网络请求代码如下

```kotlin
object ExampleRepo {

    fun getRequest1() = Observable.just(1)

    fun getRequest2() = Observable.just(2)

}
```

执行请求我们将request1的结果1和request2的结果2相加,最后结果应该为3 

```kotlin
Observable.zip(
    ExampleRepo.getRequest1(),
    ExampleRepo.getRequest2(),
    BiFunction<Int, Int, Int> { t1, t2 ->
                               t1 + t2
                              })
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(
    {
        Log.i("zhuliyuan", "onNext: $it")
    },
    {
        Log.i("zhuliyuan", "onError: $it")
    },
    {
        Log.i("zhuliyuan", "onComplete")
    },
    {
        Log.i("zhuliyuan", "onSubscribe")
    }
)
```

结果和预期相同

![](http://of1ktyksz.bkt.clouddn.com/rxjava_zipResult.png)

#### 3. 三级缓存

> 先从内存中获取数据,没获取到的话再从磁盘获取数据,还是没取到的话最后通过网络获取数据.
>
> 这里会用到`concat`操作符组合多个被观察者一起发送数据,合并后 **按发送顺序串行执行**

```kotlin
var memory: String? = null
var disk: String? = "磁盘获取数据"
var net: String? = "网络获取数据"

fun getData() {
    val memoryObservable = Observable.create<String> {
        Log.i("zhuliyuan", "从内存获取数据")
        if (!TextUtils.isEmpty(memory)) {
            it.onNext(memory!!)
        }
        it.onComplete()
    }

    val diskObservable = Observable.create<String> {
        Log.i("zhuliyuan", "从磁盘获取数据")
        if (!TextUtils.isEmpty(disk)) {
            it.onNext(disk!!)
        }
        it.onComplete()
    }.doOnNext {
        Log.i("zhuliyuan", "磁盘获取数据成功 把数据写入内存")
        memory = it
    }

    val netObservable = Observable.create<String> {
        Log.i("zhuliyuan", "从网络获取数据")
        if (!TextUtils.isEmpty(net)) {
            it.onNext(net!!)
        }
        it.onComplete()
    }.doOnNext {
        Log.i("zhuliyuan", "网络获取数据成功 把数据写入内存和磁盘")
        memory = it
        disk = it
    }

    Observable.concat(memoryObservable, diskObservable, netObservable)
    .firstElement()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(
        {
            Log.i("zhuliyuan", "onNext: $it")
        },
        {
            Log.i("zhuliyuan", "onError: $it")
        },
        {
            Log.i("zhuliyuan", "onComplete")
        }
    )
}
```

这里我模拟了三级缓存获取数据的过程,内存中没有数据所以会从磁盘获取数据,磁盘有数据,获取成功后再把数据写入内存缓存.

打印结果如下

![](http://of1ktyksz.bkt.clouddn.com/rxjava_concatResult.png)

#### 4.组合判断

> 比如填写表单数据的时候,当姓名和手机号都正确填写后提交按钮才可点击.
>
> 这里会用到`combineLatest`操作符把多个`Observable`组合在一起,将其他`Observable`的最新数据和最后一个没有发送数据的`Observable`第一次发送的数据结合在一起.

> 举个实际点的例子,有两个Observable A和B ,A发送了三个数据分别书1,2,3 然后B发送了一个数据10 那么最后就是A发送的3和B发送的10会在combineLatest 操作符中的BiFunction 对象的参数中给你,然后你来处理最后应该返回什么.

接下来我们模拟当用户姓名和手机号都正确填写后提交按钮才可点击的情况.

ui如下

```xml
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="rocketly.rxjava2demo.MainActivity">

    <TextView
        android:id="@+id/tv_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="10dp"
        android:text="姓名"
        android:textSize="18sp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <EditText
        android:id="@+id/et_name"
        android:layout_width="200dp"
        android:layout_height="32dp"
        android:layout_marginLeft="10dp"
        android:padding="0dp"
        app:layout_constraintBottom_toBottomOf="@id/tv_name"
        app:layout_constraintLeft_toRightOf="@id/tv_name"
        app:layout_constraintTop_toTopOf="@id/tv_name" />

    <TextView
        android:id="@+id/tv_phone"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="10dp"
        android:text="电话"
        android:textSize="18sp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toBottomOf="@id/tv_name" />

    <EditText
        android:id="@+id/et_phone"
        android:layout_width="200dp"
        android:layout_height="32dp"
        android:layout_marginLeft="10dp"
        android:padding="0dp"
        app:layout_constraintBottom_toBottomOf="@id/tv_phone"
        app:layout_constraintLeft_toRightOf="@id/tv_phone"
        app:layout_constraintTop_toTopOf="@id/tv_phone" />

    <Button
        android:id="@+id/bt_commit"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@color/gray"
        android:text="提交"
        app:layout_constraintTop_toBottomOf="@id/et_phone" />

</android.support.constraint.ConstraintLayout>
```

然后我们用kt的扩展方法处理下EditText让他能返回给我们需要的Observable

```kotlin
fun EditText.addTextChangedListener(): Observable<String> {
    return Observable.create<String> {
        this.addTextChangedListener(object : TextWatcher {
            override fun afterTextChanged(s: Editable?) {

            }

            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {

            }

            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
                it.onNext(s.toString())
            }

        })
    }
}
```

最后我们在用combineLatest操作符处理下这两个editText的Observable,当用户输入name不为空并且手机号输入位数满足11位,提交按钮变成可点击的绿色

```kotlin
Observable.combineLatest(
                et_name.addTextChangedListener(),
                et_phone.addTextChangedListener(),
                BiFunction<String, String, Boolean> { t1, t2 ->
                    !TextUtils.isEmpty(t1) && t2.length == 11
                })
                .subscribe {
                    if (it) {
                        bt_commit.isEnabled = true
                        bt_commit.setBackgroundColor(resources.getColor(R.color.green))
                    } else {
                        bt_commit.isEnabled = false
                        bt_commit.setBackgroundColor(resources.getColor(R.color.gray))
                    }
                }
```

然后我们看下效果,只有满足我们条件的时候按钮才可点击

![](http://of1ktyksz.bkt.clouddn.com/rxjava_combineLatestResult.gif)

#### 5. 网络请求错误重试

> 请求错误的时候重试机制.
>
> 这里用到`retryWhen`操作符,当遇到错误时,将发生的错误传递给一个新的被观察者,并决定是否需要重新订阅原始被观察者 & 发送事件

我们模拟网络请求,判断如果是io连接异常并且重试次数少于3次的话进行重试

```kotlin
var retryCount = 0

fun load(){
    Observable.error<Exception>(IOException("报错了"))
    .retryWhen {
        it.flatMap {
            if (it is IOException && retryCount < 3) {//当连接失败的时候重试
                retryCount++
                Log.i("zhuliyuan", "开始第${retryCount}次重试")
                return@flatMap Observable.just(1)
            } else {//其他错误
                return@flatMap Observable.error<Exception>(it)
            }
        }
    }
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(
        {
            Log.i("zhuliyuan", "onNext: $it")
        },
        {
            Log.i("zhuliyuan", "onError: $it")
        },
        {
            Log.i("zhuliyuan", "onComplete")
        },
        {
            Log.i("zhuliyuan", "onSubscribe")
        }
    )
}
```

日志如下

![](http://of1ktyksz.bkt.clouddn.com/rxjava_retryWhenResult.png)

#### 6.联想搜索优化

> 在搜索的时候,用户最后一次输入n秒后还未输入新的内容,这时请求接口进行联想搜索.
>
> 这里会用到`debounce`操作符,发送数据事件时,若2次发送事件的间隔＜指定时间,就会丢弃前一次的数据,直到指定时间内都没有新数据发射时才会发送后一次的数据.

我们模拟用户联想搜索这个流程,当用户最后一次输入1.5秒后还没输入新的内容,我们直接把搜索内容显示出来

ui代码如下

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="rocketly.rxjava2demo.MainActivity">

    <EditText
        android:id="@+id/et"
        android:layout_width="200dp"
        android:layout_height="32dp"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="10dp"
        android:padding="0dp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="10dp"
        android:text="搜索内容:"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/et" />

</android.support.constraint.ConstraintLayout>
```

将EditText输入监听转换成Observable用的kotlin的扩展方法

```kotlin
fun EditText.addTextChangedListener(): Observable<String> {
    return Observable.create<String> {
        this.addTextChangedListener(object : TextWatcher {
            override fun afterTextChanged(s: Editable?) {

            }

            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {

            }

            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
                it.onNext(s.toString())
            }

        })
    }
}
```

最后模拟逻辑如下

```kotlin
et.addTextChangedListener()
                .debounce(1500, TimeUnit.MILLISECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                        {
                            tv.text = "搜索内容:$it"
                            Log.i("zhuliyuan", "onNext: $it")
                        },
                        {
                            Log.i("zhuliyuan", "onError: $it")
                        },
                        {
                            Log.i("zhuliyuan", "onComplete")
                        },
                        {
                            Log.i("zhuliyuan", "onSubscribe")
                        }
                )
```

然后我们看下效果,只要两次输入间隔大于1.5秒,就会把搜索内容显示出来

![](http://of1ktyksz.bkt.clouddn.com/rxjava_debounceResult.gif)

#### 7. 防止多次点击

> 防止短时间快速点击按钮触发多次点击
>
> 这里用到`throttleFirst`操作符,在某段时间内,只发送该段时间内第1次事件

这里我们模拟快速点击的情况

ui如下

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="rocketly.rxjava2demo.MainActivity">

    <Button
        android:id="@+id/bt"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="快速点击" />

</android.support.constraint.ConstraintLayout>
```

通过kt的扩展方法封装button的点击事件,在1秒内只发送该段时间内第一次点击事件

```kotlin
fun Button.setOnClickListenerV2(listener: (View) -> Unit) {
    Observable
            .create<View> { emit ->
                this.setOnClickListener {
                    emit.onNext(it)
                }
            }
            .throttleFirst(1, TimeUnit.SECONDS)
            .subscribe {
                listener(it)
            }
}
```

然后我们设置点击代码如下,快速点击的时候只会每隔1秒触发一次点击回调

```kotlin
bt.setOnClickListener {
    Log.i("zhuliyuan", "点击了")
}
```

日志如下

![](http://of1ktyksz.bkt.clouddn.com/rxjava_throttleFirstResult.png)

可以看到即使多次点击,每次间隔也有1秒

#### 总结

> 目前结合我平常使用的就是这7个例子,如果还有更多我会持续更新,最后感谢阅读.

1. `flatMap`网络请求嵌套(A请求成功后再进行B请求的情况) 
2. `zip`网络请求合并(A请求和B请求成功后合并数据在更新ui)
3. `concat`三级缓存,从内存/磁盘/网络中读取数据
4. `combineLatest`组合判断(当A输入框和B输入框输入内容满足条件的时候Button才变成可点击的)
5. `retryWhen`网络请求出错重试
6. `debounce`联想搜索优化(用户x秒后没有再输入新的内容,进行搜索请求)
7. `throttleFirst`防止多次点击

