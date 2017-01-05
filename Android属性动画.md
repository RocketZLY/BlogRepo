###随着我对自定义view了解的深入，呆板的效果已经无法满足需要，那么为了更好的效果，动画当然是最佳的选择，Android 动画可以分为三种:view动画,帧动画,属性动画.虽然前两种使用比较简单但View animation，只能支持简单的缩放、平移、旋转、透明度基本的动画，并且只是改变了View对象绘制的位置，没有改变View对象本身，只适合不需要交互的界面,而帧动画是顺序的播放一组预先定义好的图片局限性非常大,而属性动画直接修改了view对象的属性，解决了前两个动画的不足，所以前两种动画本文略了重点就是介绍3.0以后android推出的属性动画

##**1.概述**
###属性动画顾名思义,在动画的同时更改对象的属性,并且不止可以应用于View，可以对任意对象的属性进行动画,表示一个值在一段时间内的改变，当值改变时要做什么事情完全是你自己决定的。动画默认的持续时间300ms,默认帧率10ms/帧.


###**比较常用的类:**
###ObjectAnimator,ValueAnimator,AnimatorSet其中ObjectAnimator是ValueAnimator的子类,平时最常用的也是ObjectAnimator，AnimatorSet是动画集合,具体用法后面会详细讲解


###**比较常用的属性:**
###Duration:动画持续时间
###TimeInterpolator:时间插值器,它的作用是根据时间流逝的百分比计算出插值。这样说比较抽象，比如系统提供的LinearInterpolator线性插值器说到这个大家应该知道插值器是干嘛的了。
###TypeEvaluator：估值器，根据前面TimeInterpolator算出的值结合开始值，结束值计算出当前时间的属性值。
###Frame refresh delay：帧刷新延迟，对于你的动画，多久刷新一次帧；默认为10ms，但最终依赖系统的当前状态；基本不用管。
###Repeat Count and behavoir：重复次数与方式，如播放3次、5次、无限循环，可以此动画一直重复，或播放完时再反向播放

###**整个属性值计算流程就是**：根据动画已进行的时间跟动画总时间（duration）的比计算出一个时间因子（0~1），然后TimeInterpolator根据时间因子（0~1）计算出另一个因子，最后TypeAnimator通过这个因子计算出当前时间属性值。


##**2.ObjectAnimator**
###先来看一个非常简单的例子用ObjectAnimator实现X轴方向平移
### **activity代码**
```
package com.byzk.www.animatordemo;

import android.animation.ObjectAnimator;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button bt = (Button) findViewById(R.id.bt);
        final TextView tv = (TextView) findViewById(R.id.tv);
        bt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(tv,"translationX",300);
                objectAnimator.setDuration(2000).start();


            }
        });
    }

}
```
###ObjectAnimator.ofFloat()这个方法是根据要修改属性的类型来定，由于这里translation数值为float型所以这里使用ofFloat()

###**布局文件**
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/tv"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:background="#f00" />

    <Button
        android:id="@+id/bt"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="点我" />

</LinearLayout>
```


###**效果**：
![这里写图片描述](http://img.blog.csdn.net/20160124192347539)
###是不是非常简单的实现了滑动，通过ObjectAnimator的静态工厂方法创建一个ObjectAnimator对象第一个参数是要操作的view，第二个参数是要修改的属性，第三个参数是可变数组参数，如果只设置一个参数的话，那么该属性必须有get方法，因为它表示当前值变化到设置值，而内部是通过反射机制来调用来修改对应属性值。这里有点需要注意，操作的属性必须有get，set方法（这里系统已经实现了translationX的set，get方法），不然动画无法起效，不过即使没有也没关系，有三种方法可以实现相应的效果：
###1.添加set，get方法
###2.写一个包装类并添加set，get方法去修改对应属性值
###3.用ValueAnimator实现
###很显然第一种方式我们没法实现因为我们没法修改sdk中代码所以主要方法就是2，3这里先给出第二种方式的代码，第三种方式ValueAnimator中讲解。
###**用包装类实现没有set，get方法属性动画**
```
package com.byzk.www.animatordemo;

import android.animation.ObjectAnimator;
import android.animation.ValueAnimator;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button bt = (Button) findViewById(R.id.bt);
        final TextView tv = (TextView) findViewById(R.id.tv);
        bt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                ObjectAnimator objectAnimator = ObjectAnimator.ofInt(new MyTextView(tv),"width",0,300);
                objectAnimator.setDuration(2000);
                objectAnimator.start();

            }
        });
    }
    public class MyTextView{
        private final TextView tv;

        public MyTextView(TextView tv) {
            this.tv = tv;
        }

        public void setWidth(int width){
            tv.getLayoutParams().width = width;
            tv.requestLayout();
        }

        public int getWidth(){
            return tv.getLayoutParams().width;
        }

    }
}

```



##**3.ValueAnimator**
###valueAnimator本身没有动画效果，主要通过起始值，结束值，插值器，估值器计算当前时间属性值，回调AnimatorUpdateListener接口中onAnimationUpdate方法，将当前时间属性值回调然后在设置给对应属性完成动画。
###**ValueAnimator实现没有get，set方法属性动画**

```
package com.byzk.www.animatordemo;

import android.animation.ValueAnimator;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button bt = (Button) findViewById(R.id.bt);
        final TextView tv = (TextView) findViewById(R.id.tv);
        bt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                ValueAnimator valueAnimator = ValueAnimator.ofInt(0, 300);
                valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                        int animatedValue = (int) animation.getAnimatedValue();//回调属性值
                        tv.getLayoutParams().width = animatedValue;
                        tv.requestLayout();
                    }
                });
                valueAnimator.setDuration(1000).start();

            }
        });
    }
}
```

##**4.AnimatorSet和PropertyValuesHolder**
###两个类都可以实现类似于AnimationSet的动画集的效果.
###PropertyValuesHolder：

```
PropertyValuesHolder holder = PropertyValuesHolder.ofFloat("translationX", 300);
PropertyValuesHolder holder1 = PropertyValuesHolder.ofFloat("scaleX", 0.5f);
PropertyValuesHolder holder2 = PropertyValuesHolder.ofFloat("scaleY", 2f);
ValueAnimator valueAnimator = ObjectAnimator.ofPropertyValuesHolder(tv,holder, holder1, holder2);
valueAnimator.setDuration(2000).start();
```
###AnimatorSet：
```
ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(tv, "translationX", 300);
ObjectAnimator objectAnimator1 = ObjectAnimator.ofFloat(tv, "scaleX", 0.5f);
ObjectAnimator objectAnimator2 = ObjectAnimator.ofFloat(tv, "scaleY", 2f);
AnimatorSet animatorSet = new AnimatorSet();
animatorSet.playTogether(objectAnimator,objectAnimator1,objectAnimator2);
animatorSet.setDuration(2000).start();
```
###AnimatorSet更加强大可以更加精准的控制动画的执行顺序。
```
AnimatorSet as= new AnimatorSet();
as.play(anim1).before(anim2);
as.play(anim2).with(anim3);
as.play(anim2).with(anim4)
as.play(anim5).after(amin2);
as.start();
```

##**5.xml中使用属性动画**
###跟视图动画一样属性动画也可以写在xml中

```
<?xml version="1.0" encoding="utf-8"?>
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:propertyName="translationX"
    android:valueTo="300"
    android:valueType="floatType"

    />

```
###通过AnimatorInflater去加载xml中属性动画

```
Animator animator = AnimatorInflater.loadAnimator(MainActivity.this,R.animator.animator_tv);
                animator.setTarget(tv);
                animator.start();
```

##**6.动画监听**

```
objectAnimator.addListener(new Animator.AnimatorListener() {
                    @Override
                    public void onAnimationStart(Animator animation) {

                    }

                    @Override
                    public void onAnimationEnd(Animator animation) {

                    }

                    @Override
                    public void onAnimationCancel(Animator animation) {

                    }

                    @Override
                    public void onAnimationRepeat(Animator animation) {

                    }
                });
```
###除了上面这种实现所有方法的，android还给我们提供了一个AnimatorListenerAdapter，即AnimatorListener接口的空实现，需要什么的时候就重写什么方法完成监听

##**7.TimeInterpolator和TypeEvaluator**
###一般情况下插值器和估值器使用系统自带的即可完成大部分任务，如果实在需要自定义炫酷的效果可以重新实现插值器和估值器

```
objectAnimator.setInterpolator(new TimeInterpolator() {
                    @Override
                    public float getInterpolation(float input) {
                        return 0;//插值器的实现
                    }
                });
objectAnimator.setEvaluator(new TypeEvaluator() {
                    @Override
                    public Object evaluate(float fraction, Object startValue, Object endValue) {
                        return null;//估值器的实现
                    }
                });
```

##**8.View的anim方法**
###为了方便使用动画还可以view.animate()这样写来

```
iv.animate().alpha(0.5f)
                .setDuration(2000)
                .setListener(new AnimatorListenerAdapter() {

                })
                .start();
```

###到这里属性动画的基本使用就讲解完了以后还会写一些利用属性动画实现好看效果的自定义控件。
