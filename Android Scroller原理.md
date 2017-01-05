##一般说到弹性滑动,我们都会想到Scroller这个类,虽然用的非常多但可能平时并未去分析它是如何实现的,而本文先从Scroller平时的使用到源码的角度分析一下Scroller是如何实现.

##先列出Scroller的使用
```
//1.构建对象
Scroller scroller = new Scroller(context)

//2.重写View的computeScroll()方法
@Override
public void computeScroll() {
    if(scroller.computeScrollOffset()){
        scrollTo(scroller.getCurrX(),scroller.getCurrY());
        invalidate();
    }
}

//3.开启Scroller滑动
scroller.startScroll(0,0,100,0);
invalidate();
```

##下面再来从源码的角度来分析如何实现弹性滑动的
##先看下Scroller的startScroll()源码

```
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }
```
###从上面源码看出其实调用startScroller()时候什么也没做只是记录了一些数值,所以仅仅调用该方法是无法让View滑动的,注意这里的滑动指的是View内容的滑动而非View位置的改变,那么能让view滑动的只剩下invalidate(),invalidate方法会导致view的重绘,而view的draw方法又会调用computeScroll方法,computeScroll()方法上面已经实现即获取当前Scroller的scrollX和ScrollY,通过scrollTo()实现滑动并且最后还调用了invalidata()形成递归直到最后滑动结束.

##在看下Scroller的computeScrollOffset()源码

```
public boolean computeScrollOffset() {
        ....


        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);

        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE:
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;

				....
            }

	     ....
        return true;
    }
```
###这里只截取了有用的一部分,先看timePassed为当前时间减去刚才startScroll()时候记录的时间mStartTime得出当前调用computeScrollOffset()和startScroll()时间间隔,如果小于duration继续往下,根据时间间隔占duration的百分比计算当前应滑动到的位置,每次滑动会有一个小的时间间隔,再在computeScroll()实现方法中scrollTo(scroller.getCurrX(),scroller.getCurrY())实现View每次的小幅度滑动来实现弹性滑动,而computeScrollOffset()的返回值ture代表滑动未执行完,false反之.
