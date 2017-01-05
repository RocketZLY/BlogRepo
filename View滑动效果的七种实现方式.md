>关于android中可以滑动的控件可以说数不胜数,而同一个效果的实现方式也是非常多的,今天先用最简单的几种方式来感受下自定义控件滑动效果的产生,先上效果图

![](http://img.blog.csdn.net/20151230165849302)

布局都是一样的
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.byzk.www.viewdemo.MainActivity">


    <com.byzk.www.viewdemo.DragView
        android:id="@+id/dv"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:background="#000" />


</LinearLayout>

```
下面来看不同的实现方式

# 1.根据layout()方法来实现
根据触摸事件记录按下坐标和移动的坐标计算偏移量,通过偏移量来改变view的坐标,不断重复实现滑动
而说到计算获取手指按下的坐标先看一张图了解下如下几个方法

![](http://img.blog.csdn.net/20151230175432270)

- getTop()获取到的是View自身顶边到父布局顶边的距离

- getBottom()获取到的是View自身底边到父布局顶边的距离

- getLeft()获取到的是View自身左边到父布局左边的距离

- getRight()获取到的是View自身右边到父布局左边的距离

- MotionEvent中的方法

- getX()获取点击位置距离当前View左边的距离

- getY()获取点击位置距离当前View上边的距离

- getRawX()获取点击位置距离屏幕左边的距离

- getRawY()获取点击位置距离屏幕上边的距离

```
package com.byzk.www.viewdemo;

import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;

/**
 * Author: zhuliyuan
 * Time: 下午 2:47
 */

public class DragView extends View {


    public DragView(Context context) {
        this(context, null);
    }

    public DragView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DragView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
    }


    private int startX;
    private int startY;

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int x = (int) event.getX();//表示相对于当前View的x
        int y = (int) event.getY();//表示相对于当前View的y
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                startX = x;
                startY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                int distanceX = x - startX;
                int distanceY = y - startY;
                layout(
                        getLeft()+distanceX,
                        getTop()+distanceY,
                        getRight()+distanceX,
                        getBottom()+distanceY
                );
                break;
            case MotionEvent.ACTION_UP:


                break;
        }
        return true;
    }


}

```
或者使用MotionEvent的getRawX(),getRawY()获取点击的坐标,但是要注意使用的这种绝对坐标系,一定要重新初始化初始坐标

# 2.offsetLeftAndRight(),offsetTopAndBottom()实现
跟上面效果一样,计算出偏移量后只需要下面代码即可

```
offsetLeftAndRight(distanceX);
offsetTopAndBottom(distanceY);
```
# 3.LayoutParams实现
计算出偏移量后

```
LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) getLayoutParams();
lp.topMargin = getTop()+distanceY;
lp.leftMargin = getLeft() + distanceX;
setLayoutParams(lp);
```
# 4.scrollTo或者scrollBy实现
scrollTo()移动到某点,scrollBy()移动的偏移量

这两个方法与上面的有些不同,他移动的为View的内容的位置,而不是改变View的位置,并且当View内容左边缘在View左边缘左边时候为正,View内容上边缘在View上边缘上面的时候为正,也就是说不管怎么滑动,当前View不可能滑动到别的View的附近,听起来可能有点绕,画张图来理解下

![](http://img.blog.csdn.net/20151231103948890)

![](http://img.blog.csdn.net/20151231103252244)

说白了就是把View位置看成固定的,内容移动,并且他的坐标系正好跟我们平时的坐标系相反
那么要实现跟上面相似的功能,只需要将move事件中代码改为如下

```
int distanceX = x - startX;
int distanceY = y - startY;
((View)getParent()).scrollBy(-distanceX,-distanceY);
```

# 5.Scroller实现
Scroller是将一次较长的滑动,按一定时间分成多次较小的滑动实现视觉上的弹性滑动,但是代码会稍微多一点
在上面代码基础上实现松开手指后的回弹到原点的弹性滑动效果效果图如下

![](http://img.blog.csdn.net/20151231113840398)

代码如下
```
package com.byzk.www.viewdemo;

import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;
import android.widget.Scroller;
import android.widget.TextView;

/**
 * Author: zhuliyuan
 * Time: 下午 2:47
 */

public class DragView extends TextView {


    private Scroller scroller;

    public DragView(Context context) {
        this(context, null);
    }

    public DragView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DragView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
        scroller = new Scroller(context);
    }

    @Override
    public void computeScroll() {
        super.computeScroll();
        if(scroller.computeScrollOffset()){
            ((View)getParent()).scrollTo(scroller.getCurrX(),scroller.getCurrY());
            invalidate();//这里不断的递归修改坐标形成视觉上的弹性滑动
        }
    }

    private int startX;
    private int startY;

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int x = (int) event.getX();//表示相对于当前view的x
        int y = (int) event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                startX = x;
                startY = y;
                break;
            case MotionEvent.ACTION_MOVE:
                int distanceX = x - startX;
                int distanceY = y - startY;
                ((View)getParent()).scrollBy(-distanceX, -distanceY);



                break;
            case MotionEvent.ACTION_UP:
                View parent = (View) getParent();
                scroller.startScroll(
	                parent.getScrollX(),
	                parent.getScrollY(),
	                -parent.getScrollX(),
	                -parent.getScrollY());
                invalidate();//这里调用了就会触发ondraw()然后会调用computeScroll()方法
                break;
        }
        return true;
    }


}

```
# Scroller使用

1. 创建对象

2. 重写computeScroll()代码

3. startScroll()开始模拟过程

如果还想知道Scroller源码中是如何实现的我的另外篇博客可以帮到你
http://blog.csdn.net/zly921112/article/details/50524852

# 6.属性动画实现
对于属性动画不是很清楚的小伙伴可以先看下我写的关于属性动画的使用
http://blog.csdn.net/zly921112/article/details/50555332

效果

![](http://img.blog.csdn.net/20160124192347539)

activity代码

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
布局文件

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

到此简单的滑动方式就说完啦.还有ViewDragHelper这种复杂点的方式将在后面的帖子中再次讲解.
