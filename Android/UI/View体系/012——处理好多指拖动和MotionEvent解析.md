# 1，简单的写一个拖动控件


为了理解这个知识点，首先写一个没有处理多指拖动的DragLayout。代码很简单，就是实现一个可垂直拖动子view的布局。

代码如下，非常的简单。


    public class DragLayout extends FrameLayout {

        private static final String TAG = DragLayout.class.getSimpleName();
        private float mLastX;//手指在屏幕上最后的x位置
        private float mLastY;//手指在屏幕上最后的y位置
        private float mDownX;//手指第一次落下时的x位置（忽略）
        private float mDownY;//手指第一次落下时的y位置

        private int mScaledTouchSlop;//认为是滑动行为的最小参考值
        private boolean mIntercept;//是否拦截事件

        public DragLayout(Context context) {
            this(context, null);
        }

        public DragLayout(Context context, AttributeSet attrs) {
            this(context, attrs, 0);
        }

        public DragLayout(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            init();
        }

        private void init() {
            mScaledTouchSlop = ViewConfiguration.get(getContext()).getScaledTouchSlop();
        }


        @Override
        public boolean onInterceptTouchEvent(MotionEvent ev) {

            float x = ev.getX();
            float y = ev.getY();
            int action = ev.getAction();


            switch (action) {
                case MotionEvent.ACTION_DOWN: {
                    mIntercept = false;
                    mLastX = x;
                    mLastY = y;

                    mDownX = x;
                    mDownY = y;
                    break;
                }

                case MotionEvent.ACTION_MOVE: {

                    if (!mIntercept) {//没有没有拦截，才去判断是否需要拦截
                        float offset = Math.abs(mDownY - y);
                        Log.d(TAG, "offset:" + offset);
                        if (offset >= mScaledTouchSlop) {
                            float dx = mLastX - x;
                            float dy = mLastY - y;
                            if (Math.abs(dy) > Math.abs(dx)) {
                                mIntercept = true;
                            }
                        }
                    }

                    break;
                }

                case MotionEvent.ACTION_UP: {
                    mIntercept = false;
                    break;
                }
            }

            mLastX = x;
            mLastY = y;

            return mIntercept;
        }


        @Override
        public boolean onTouchEvent(MotionEvent ev) {
            float x = ev.getX();
            float y = ev.getY();

            int action = ev.getAction();

            switch (action) {
                case MotionEvent.ACTION_DOWN: {

                    mLastX = x;
                    mLastY = y;

                    break;
                }

                case MotionEvent.ACTION_MOVE: {

                    float dy = mLastY - y;
                    scrollBy(0, (int) dy);

                    break;
                }

                case MotionEvent.ACTION_UP: {
                    break;
                }
            }

            mLastX = x;
            mLastY = y;

            return true;
        }
    }

效果图如下：

![](img/012_bad_event.gif)


但是有没有发现一些不好的地方呢？
当第一个手指往下拖动了一下控件，接着第二个手指也触摸了屏幕，然后第一个离开了屏幕，这时你会看到红色的子view往上跳动了一下，这个跳动实在是太突兀了，我们希望的应该是当第一个手指离开屏幕时，红色的子view不会有任何跳动，而是依然顺畅的被第二个手指继续拖动。


如下面图所示：

![](img/012_good_event.gif)




把拖动变得顺畅需要处理多指拖动的情况，而要处理好多指拖动的情况，则需要了解MotionEvent类


# 2，MotionEvent解析

用户在屏幕上的每个触摸行为都会被当成一个事件，而这个事件肯定有一系列属性，比如事件发送的**位置**，**类型**等等。

而在Android中描述这个触摸事件的就是MotionEvent，与事件分发机制相关的方法都接受一个MotionEvent对象，比如`dispatchTouchEvent`，`onInterceptTouchEvent` ，`onTouchEvent`，MotionEvent常用的属性和方法：



### 1：获取事件的位置

    getX()      获取事件的x坐标，相对于当前View
    getY()      获取事件的y坐标，相对于当前View
    getRawX()   获取事件的x坐标,相对于屏幕
    getRawY()   获取事件的y坐标,相对于屏幕



getX()表示获取事件相对于当前View区域的x方向的位置，那么这个值是如何得到的呢，它的处理在ViewGroup的dispatchTransformedTouchEvent方法中，父View把自身的scroll值减去子view的left，即得到事件相对于子view区域的位置。同理getY()也是如何。

        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        event.offsetLocation(offsetX, offsetY);
        handled = child.dispatchTouchEvent(event);

### 2：事件的index和pointId

    getActionIndex()          获取事件的索引值
    getPointerId(int index)   根据事件索引获取事件的id
    getPointerCount()         获取触摸点的个数



由于android系统是支持多只触控的，所以在同一时刻可能会有多个触摸点，而pointerId和actionIndex就是为了区分这些触摸点。

- pointerId表示一个触摸点的id，它的特点是在触摸点触摸屏幕的那一刻被赋值，直到触摸点立刻屏幕之前，这个触摸点对应的pointId都不会改变。

- actionIndex表示一个触摸点的索引，它总是从0开始而且随着触摸点的个数而动态改变。当有两个触摸点时，其中一个触摸点的索引必然是0，另一个必然是1.

一个MotionEvent可能会包含多个触摸点的信息。而通过pointerId和index我们可以获取不同触摸点的信息。比如：

    首先通过getActionIndex获取触摸的索引
    getPointerId(int pointerIndex)  ：通过index获取触摸点的Id
    findPointerIndex(int pointerId) :通过id获取触摸点的index
    getX(int pointerIndex) ：通过index获取对应触摸点的x坐标
    getY(int pointerIndex) ：通过index获取对应触摸点的y坐标



### 3：getAction与getActionMasked

    getAction()               获取事件的类型，这是一个组合值，由pointer的index值和事件类型值组合而成的
    getActionMasked()         获取事件的类型，不具有其他信息


getAction获取的是一个组合值而getActionMasked获取的值仅仅表示当前事件的类型。

那么这是是如何计算的呢？需要看一下源码：

    int ACTION_MASK   = 0xff;        //位遮罩，二进制位 1111 1111
    int ACTION_DOWN   = 0;
    int ACTION_UP     = 1;
    int ACTION_MOVE   = 2;


    //获取组合值
    public final int getAction() {
            return nativeGetAction(mNativePtr);
     }
    //获取事件类型值
     public final int getActionMasked() {
            return nativeGetAction(mNativePtr) & ACTION_MASK;
     }

    int ACTION_POINTER_INDEX_MASK  = 0xff00;
    int ACTION_POINTER_INDEX_SHIFT = 8;
    //获取事件的索引值
    public final int getActionIndex() {
            return (nativeGetAction(mNativePtr) & ACTION_POINTER_INDEX_MASK)
                    >> ACTION_POINTER_INDEX_SHIFT;
    }

getAction通过和 `ACTION_MASK `位于运算得到纯类型的值，ACTION_MASK的二进制表示为`1111 1111`，这个位于运算是为了清除Action组合值前8位的信息，由此得到事件的类型值由int值的最后8位表示。

再看获取`getActionIndex`的算法，通过组合值位于`ACTION_POINTER_INDEX_MASK`，再向右移动8位，所以我们可以得出结论，事件的索引值和pointId由int值的前8位表示。

**当只有单个触摸点时，getAction和getActionMasked获取的值是一样的**




#### 4 事件类型

    ACTION_DOWN            表示第一个手指按下
    ACTION_UP              表示最后一个手指离开屏幕
    ACTION_MOVE            表示一个触摸点移动事件
    ACTION_POINTER_DOWN    表示一个非主要手指按下(必然已至少存在一个触摸点)
    ACTION_POINTER_UP      表示一个非主要手指离开屏幕(必然至少还存在一个触摸点)
    ACTION_CANCEL          表示一个事件被取消
    ACTION_OUTSIDE         表示手势操作发生在UI组件外

上面列举了一些常用的事件类型，而且已经注释的非常清楚了，下面对`ACTION_CANCEL`做一下特别说明。

ACTION_CANCEL发送的场景：比如说在一个完整的事件系列中父控件首先不拦截事件而让子view可以获取和处理事件，而在某一个时刻父控件又开始拦截事件，这时子view的事件将会被中断，以一个ACTION_CANCEL结尾。

可能还有一点不明白，为啥子有`ACTION_POINTER_DOWN`和`ACTION_POINTER_UP`，却没有`ACTION_POINTER_MOVE`呢？


这是因为考虑到触摸点移动事件发生的频率非常高，哪怕移动一小段距离也会产生很多个MOVE事件，所以为了效率和内存，安卓系统把连续的几个多触点移动事件打包到一个`MotionEvent`对象中。而前面我们也说到MotionEvent可以包含多个触摸点事件的信息。通过`getX(int)`和`getY(int)`来获得的值表示**最近发生的一个触摸点事件**的坐标。
这时我们需要通过`getHistoricalXXX`系列方法来获取时间上稍早的触点事件的信息。

在[官方文档中](http://developer.android.com/intl/zh-cn/reference/android/view/MotionEvent.html)有如下一段代码,表示如何处理这种事件类型：

    void printSamples(MotionEvent ev) {
            //返回此事件中的历史点数
            final int historySize = ev.getHistorySize();
            //返回事件表示的触摸点个数
            final int pointerCount = ev.getPointerCount();
            Log.d(TAG + "his", "historySize:" + historySize);
            Log.d(TAG + "his", "pointerCount:" + pointerCount);
            for (int h = 0; h < historySize; h++) {
                Log.d(TAG + "his", "ev.getHistoricalEventTime(h):" + ev.getHistoricalEventTime(h));
                for (int p = 0; p < pointerCount; p++) {
                    Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getPointerId(p));
                    Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getHistoricalX(p, h));
                    Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getHistoricalY(p, h));

                }
            }
            Log.d(TAG, "ev.getEventTime():" + ev.getEventTime());
            for (int p = 0; p < pointerCount; p++) {
                Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getPointerId(p));
                Log.d(TAG + "his", "ev.getX(p):   and    ev.getY(p):" + ev.getX(p) + "   " + ev.getY(p));
            }
        }

做了一下小改动，System.out.println()打不出log，**getHistoricalXXX只适用于MOVE事件**

## MotionEventCompat

使用v4包中的MotionEventCompat可以帮助我们更好的兼容各种API版本。




# 处理多指拖动

分析完MotionEvent,再来思考一下如何处理多指拖动。思路是这样的：
1，一个触摸点的pointerId在离开屏幕之前是不会改变的
2，我们在处理拖动的时候首先确认好一个pointerId,然后根据此pointerId获取对应的触摸点的位置信息，也就是我们同一时刻值处理一个触摸点
3，当有一个新的手指按下（一个新的触摸点产生时），我们需要切换我们关心的pointerId，这是我们的处理对象就发生变化了，而此时为了防止子View的跳动，我们同时还需要更新触摸点的y值。
4，当有一个主要的手指抬起时，我们判断这个抬起的手指是不是我们当前正在关心的那个pointerId对于的手指(触摸点)，如果是我们的处理还是更新pointerId和y值。

代码实现如下：


    package com.ztiany.mydemo.view;

    import android.content.Context;
    import android.support.v4.view.MotionEventCompat;
    import android.util.AttributeSet;
    import android.util.Log;
    import android.view.MotionEvent;
    import android.view.ViewConfiguration;
    import android.widget.FrameLayout;

    /**
     * @author Ztiany
     *         email 1169654504@qq.com & ztiany3@gmail.com
     *         date 2016-03-21 15:33
     *         description
     *         vsersion
     */
    public class MultiDragLayout extends FrameLayout {

        private static final String TAG = MultiDragLayout.class.getSimpleName();
        private float mLastX;
        private float mLastY;
        private float mDownX;//test
        private float mDownY;

        public static final int INVALID_POINTER = MotionEvent.INVALID_POINTER_ID;
        private int mActivePointerId = MotionEvent.INVALID_POINTER_ID;

        private int mScaledTouchSlop;
        private boolean mIntercept;

        public MultiDragLayout(Context context) {
            this(context, null);
        }

        public MultiDragLayout(Context context, AttributeSet attrs) {
            this(context, attrs, 0);
        }

        public MultiDragLayout(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            Log.d(TAG, "MultiDragLayout() called with: " + "context = [" + context + "], attrs = [" + attrs + "], defStyleAttr = [" + defStyleAttr + "]");
            init();
        }

        private void init() {
            mScaledTouchSlop = ViewConfiguration.get(getContext()).getScaledTouchSlop();
            Log.d(TAG, "mScaledTouchSlop:" + mScaledTouchSlop);
        }


        @Override
        public boolean onInterceptTouchEvent(MotionEvent ev) {
            int action = MotionEventCompat.getActionMasked(ev);

            switch (action) {
                case MotionEvent.ACTION_DOWN: {
                    //重置拦截标识
                    mIntercept = false;
                    //获取初始的位置
                    mLastX = ev.getX();
                    mLastY = ev.getY();
                    mDownX = mLastX;
                    mDownY = mLastY;
                    //这里我们根据最初的触摸的确定一个pointerId
                    int index = ev.getActionIndex();
                    mActivePointerId = ev.getPointerId(index);

                    break;
                }
                case MotionEventCompat.ACTION_POINTER_DOWN: {

                    Log.d(TAG, "onInterceptTouchEvent() called with: " + "ev = [ACTION_POINTER_DOWN  ]");

                    break;
                }
                case MotionEvent.ACTION_MOVE: {
                    //如果我们关系的pointerId==-1，不再拦截
                    if (mActivePointerId == INVALID_POINTER) {
                        return false;
                    }

                    final int pointerIndex = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
                    if (pointerIndex < 0) {
                        return false;
                    }

                    float currentY = MotionEventCompat.getY(ev, pointerIndex);
                    float currentX = MotionEventCompat.getX(ev, pointerIndex);

                    if (!mIntercept) {
                        float offset = Math.abs(mDownY - currentY);
                        Log.d(TAG, "offset:" + offset);
                        if (offset >= mScaledTouchSlop) {
                            float dx = mLastX - currentX;
                            float dy = mLastY - currentY;
                            if (Math.abs(dy) > Math.abs(dx)) {
                                mIntercept = true;
                            }

                        }
                    }
                    mLastX = currentX;
                    mLastY = currentY;
                    break;
                }
                case MotionEventCompat.ACTION_POINTER_UP: {

                    Log.d(TAG, "onInterceptTouchEvent() called with: " + "ev = [ACTION_POINTER_UP  ]");
                    //处理手指抬起
                    onSecondaryPointerUp(ev);
                    break;
                }
                case MotionEvent.ACTION_UP: {
                    mIntercept = false;
                    mActivePointerId = INVALID_POINTER;
                    break;
                }
            }


            return mIntercept;
        }

        private void onSecondaryPointerUp(MotionEvent ev) {
            int index = MotionEventCompat.getActionIndex(ev);
            int pointerId = MotionEventCompat.getPointerId(ev, index);
            if (mActivePointerId == pointerId) {//如果是主要的手指抬起
                final int newPointerIndex = index == 0 ? 1 : 0;//确认一个还在屏幕上手指的index
                mLastY = MotionEventCompat.getY(ev, newPointerIndex);//更新lastY的值
                mActivePointerId = MotionEventCompat.getPointerId(ev, newPointerIndex);//更新pointerId
            }
        }


        private void onSecondaryPointerDown(MotionEvent ev) {
            final int index = MotionEventCompat.getActionIndex(ev);
            mLastY = MotionEventCompat.getY(ev, index);
            mActivePointerId = MotionEventCompat.getPointerId(ev, index);
        }


        @Override
        public boolean onTouchEvent(MotionEvent ev) {
            int action = MotionEventCompat.getActionMasked(ev);

            switch (action) {
                case MotionEvent.ACTION_DOWN: {

                    mLastX = ev.getX();
                    mLastY = ev.getY();
                    mDownX = mLastX;
                    mDownY = mLastY;
                    int index = ev.getActionIndex();
                    mActivePointerId = ev.getPointerId(index);

                    break;
                }
                case MotionEvent.ACTION_POINTER_DOWN: {
                    Log.d(TAG, "onTouchEvent() called with: " + "ev = [ACTION_POINTER_DOWN  ]");
                    onSecondaryPointerDown(ev);
                    break;
                }
                case MotionEvent.ACTION_MOVE: {
                    printSamples(ev);
                    int pointerIndex = MotionEventCompat.findPointerIndex(ev, mActivePointerId);//主手指的索引
                    if (pointerIndex < 0) {
                        return false;
                    }

                    float currentX = MotionEventCompat.getX(ev, pointerIndex);
                    float currentY = MotionEventCompat.getY(ev, pointerIndex);
                    int dy = (int) (mLastY - currentY);
                    scrollBy(0, dy);

                    mLastX = currentX;
                    mLastY = currentY;
                    break;
                }
                case MotionEvent.ACTION_POINTER_UP: {
                    Log.d(TAG, "onTouchEvent() called with: " + "ev = [ACTION_POINTER_UP  ]");
                    onSecondaryPointerUp(ev);
                    break;
                }
                case MotionEvent.ACTION_UP: {
                    mIntercept = false;
                    mActivePointerId = INVALID_POINTER;
                    break;
                }
            }


            return true;
        }

        //for test
        void printSamples(MotionEvent ev) {
            final int historySize = ev.getHistorySize();
            final int pointerCount = ev.getPointerCount();
            Log.d(TAG + "his", "historySize:" + historySize);
            Log.d(TAG + "his", "pointerCount:" + pointerCount);
            for (int h = 0; h < historySize; h++) {
                Log.d(TAG + "his", "ev.getHistoricalEventTime(h):" + ev.getHistoricalEventTime(h));
                for (int p = 0; p < pointerCount; p++) {
                    Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getPointerId(p));
                    Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getHistoricalX(p, h));
                    Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getHistoricalY(p, h));

                }
            }
            Log.d(TAG, "ev.getEventTime():" + ev.getEventTime());
            for (int p = 0; p < pointerCount; p++) {
                Log.d(TAG + "his", "ev.getPointerId(p):" + ev.getPointerId(p));
                Log.d(TAG + "his", "ev.getX(p):   and    ev.getY(p):" + ev.getX(p) + "   " + ev.getY(p));
            }
        }


    }
