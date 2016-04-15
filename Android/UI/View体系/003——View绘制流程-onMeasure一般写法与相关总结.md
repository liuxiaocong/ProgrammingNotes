# 1 View的onMeasure一般写法

      private Bitmap mBitmap;

        private void init() {
            mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.meinv);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            // 获取宽度测量规格中的mode和size
            int widthMode = MeasureSpec.getMode(widthMeasureSpec);
            int widthSize = MeasureSpec.getSize(widthMeasureSpec);
            int heightMode = MeasureSpec.getMode(heightMeasureSpec);
            int heightSize = MeasureSpec.getSize(heightMeasureSpec);
            // 声明一个临时变量来存储计算出的测量值
            int  heightResult = 0;
            int widthResult = 0;

             /*
             * 如果父容器心里有数
             */
            if (widthMode == MeasureSpec.EXACTLY) {
                // 那么子view就用父容器给定尺寸
                widthResult = widthSize;
            }
            /*
             * 如果父容器不确定自身大小
             */
            else{
                // 那么子view可要自己看看自己需要多大了
                widthResult = mBitmap.getWidth()+getPaddingLeft()+getPaddingRight();
                /*
                   * 如果爹给儿子的是一个限制值
                 */
                if (widthMode == MeasureSpec.AT_MOST) {
                    // 那么儿子自己的需求就要跟爹的限制比比看谁小要谁
                    widthResult = Math.min(widthSize, widthResult);
                }
            }

            if (heightMode == MeasureSpec.EXACTLY) {
                heightResult = widthSize;
            }else{
                //考虑padding
                heightResult = mBitmap.getHeight()+getPaddingBottom()+getPaddingTop();
                if (heightMode == MeasureSpec.AT_MOST) {
                    heightResult = Math.min(widthSize, heightResult);
                }
            }
            // 设置测量尺寸
            setMeasuredDimension(widthResult, heightResult);
        }



流程就是：
1. 获取测量尺寸和模式,定义临时变量存储技术结果

2. 判断测量模式：
 - 模式是EXACTL的，就使用测量规格中的尺寸
 - 模式是UNSPECIFIED，使用自身计算的尺寸
 - 模式是AT_MOST的，使用自身计算的尺寸与规定尺寸中较小的一个

3.设置测量尺寸




##ViewGroup的onMeasure一般写法

由于ViewGroup的布局卡变万化，根本没有统一的模板，只能根据业务来定，一般大概流程就是：

1. 首先ViewGroup作为容器，有测量所有子view的职责，所以第一步是遍历测量所有的字view。一般我们会选用ViewGroup提供的`measureChildWithMargins`方法，这个方法已经考虑了子view需要的margin和ViewGroup自身需要的padding。
2. 遍历测量完所有子view，那么子view的宽高基本可以确定了，如果有特殊的需求，可以对子view进行多次测量。
3. 根据自身的布局特性，来确定自身的宽高，但是需要遵守基本的测量规则



对于定制化的ViewGroup当然据根据业务需求来定了。


<br/><br/>



下面是爱哥博客中的一个demo，仅供参考(没有完成)：

```java

/**
 * Author Ztiany      <br/>
 * Email ztiany3@gmail.com      <br/>
 * Date 2015-10-03 10:51      <br/>
 * Description：测量练习
 * <p/>
 * 练习：实现一个自定义布局：里面所有的子view按照正方形的样式排列，允许定义多行多列
 */
public class SquareEnhanceLayout extends ViewGroup {
    private static final String TAG = SquareEnhanceLayout.class.getSimpleName();
    private int childMeasureState;

    public SquareEnhanceLayout(Context context) {
        this(context, null);
    }

    public SquareEnhanceLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public SquareEnhanceLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
        TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.ViewGroupMeasureDemoView);
        mOrientation = array.getInt(R.styleable.ViewGroupMeasureDemoView_orientation, 0);

        mMaxColumn = array.getInteger(R.styleable.ViewGroupMeasureDemoView_column, 1);
        mMaxRow = array.getInteger(R.styleable.ViewGroupMeasureDemoView_row, 1);

            Log.d("ViewGroupMeasureDemoVie", "orientation:" + mOrientation);
        array.recycle();
    }

    private int mOrientation;//排列方向
    public final int ORIENTATION_VERTICAL = 0;//横向
    public final int ORIENTATION_HRIOZONTAL = 1;//纵向

    public static final int DEFAULT_MAX_ROW = Integer.MAX_VALUE, DEFAULT_MAX_COLUMN = Integer.MAX_VALUE;
    public int mMaxRow = DEFAULT_MAX_ROW;//最大行数
    public int mMaxColumn = DEFAULT_MAX_COLUMN;//最大列数

    //
    private void init() {
// 初始化最大行列数
        mMaxRow = mMaxColumn = 1;
    }


    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 声明临时变量存储父容器的期望值
        int parentDesireWidth = 0;
        int parentDesireHeight = 0;
        //获取子view的个数
        int count = getChildCount();

        View child = null;
        if (count > 0) {//有孩子采取测量

            // 声明两个一维数组存储子元素宽高数据
            int[] childWidths = new int[getChildCount()];
            int[] childHeights = new int[getChildCount()];


            for (int i = 0; i < count; i++) {
                child = getChildAt(i);
                if (child.getVisibility() == View.GONE) {//如果view不可见，就不进行测量了


                    continue;
                }

                CustomLayoutParams customLayoutParams = (CustomLayoutParams) child.getLayoutParams();
                //对子view进行测量 并且考虑布局margin
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                //获取子view测量后的宽高指中的最大宽高值（这里需要子view都是按照正方形显示的）
                int childMaxSize = Math.max(child.getMeasuredHeight(), child.getMeasuredWidth());
                //取最大值，让子view重新测量
                int childMeasureSpec = MeasureSpec.makeMeasureSpec(childMaxSize, MeasureSpec.EXACTLY);
                child.measure(childMeasureSpec, childMeasureSpec);

                  /*
                     * 考量外边距计算子元素实际宽高并将数据存入数组
                     */
                childWidths[i] = child.getMeasuredWidth() + customLayoutParams.leftMargin + customLayoutParams.rightMargin;
                childHeights[i] = child.getMeasuredHeight() + customLayoutParams.topMargin + customLayoutParams.bottomMargin;

                // 合并子元素的测量状态 关于测量状态也只是个实验性功能，按照谷歌官方做即可
                childMeasureState = combineMeasuredStates(childMeasureState, child.getMeasuredState());

            }


            // 声明临时变量存储行/列宽高
            int indexMultiWidth = 0, indexMultiHeight = 0;

            //父view根据孩子的测量结果，来计算自己的大小
            if (mOrientation == ORIENTATION_HRIOZONTAL) {//横
                //如果子view的个数大于最大列数，说明需要换行
                if (count > mMaxColumn) {

                    int row = count / mMaxColumn;//行数
                    int remainder = count & mMaxColumn;//余数
                    // 声明临时变量存储子元素宽高数组下标值
                    int index = 0;

                    for (int x = 0; x < row; x++) {

                        for (int y = 0; y < mMaxColumn; y++) {//为什么是 x<row y<mMaxColumn  因为是横布局，整体一行一行来测量，一行中一个一个来测量
                            //单行累加宽度
                            indexMultiWidth += childWidths[index];
                            //高度取最大值
                            indexMultiHeight = Math.max(indexMultiHeight, childHeights[index]);
                            index++;
                        }
                        //一行遍历完毕后 高度累加 宽度取最大值
                        parentDesireHeight += indexMultiHeight;
                        parentDesireWidth = Math.max(parentDesireWidth, indexMultiWidth);
                        //一行遍历完毕 重置
                        indexMultiHeight = 0;
                        indexMultiWidth = 0;
                    }
                        /*
                     * 如果有余数表示有子元素未能占据一行
                     */
                    if (remainder > 0) {
                        for (int i = count - remainder; i < count; i++) {
                            indexMultiHeight = Math.max(indexMultiHeight, childHeights[i]);
                            indexMultiWidth += childWidths[i];
                        }
                        //最后一行遍历完毕后 高度累加 宽度取最大值
                        parentDesireHeight += indexMultiHeight;
                        parentDesireWidth = Math.max(parentDesireWidth, indexMultiWidth);
                        //一行遍历完毕 重置
                        indexMultiHeight = 0;
                        indexMultiWidth = 0;
                    }

                } else {
                    //没有列数限制，就是一行
                    for (int x = 0; x < count; x++) {
                        //横向的布局 横向累加，要考虑子view的margin
                        parentDesireWidth += childWidths[x];
                        parentDesireHeight = Math.max(parentDesireHeight, childHeights[x]);
                    }
                }

            } else if (mOrientation == ORIENTATION_VERTICAL) {//纵向


            }

            //下面四段逻辑是 在 count>0之内

            //最后要考虑自身的padding
            parentDesireWidth += getPaddingLeft() + getPaddingRight();
            parentDesireHeight += getPaddingTop() + getPaddingBottom();
             /*
             * 尝试比较父容器期望值与Android建议的最小值大小并取较大值
             */
            parentDesireWidth = Math.max(getSuggestedMinimumWidth(), parentDesireWidth);
            parentDesireHeight = Math.max(getSuggestedMinimumHeight(), parentDesireHeight);

        }

         /*
           那么这个resolveSize方法其实是View提供给我们解算尺寸大小的一个工具方法，
           其具体实现在API 11后交由另一个方法resolveSizeAndState也就是我们这一节例子所用到的去处理：
            这个方法多了一个childMeasuredState参数，而上面例子我们在具体测量时也引入了一个childMeasureState临时变量的计算，
            那么这个值的作用是什么呢？有何意义呢？说到这里不得不提API 11后引入的几个标识位：
            MEASURED_HEIGHTSTATE_SHIFT，MEASURED_SIZE_MASK，MEASURED_STATE_MASK,MEASURED_STATE_TOO_SMALL

            “childMeasuredState这个值呢由View.getMeasuredState()这个方法返回，一个布局（或者按我的说法父容器）
            通过View.combineMeasuredStates()这个方法来统计其子元素的测量状态。在大多数情况下你可以简单地只传递0作为参数值，
            而子元素状态值目前的作用只是用来告诉父容器在对其进行测量得出的测量值比它自身想要的尺寸要小，
            如果有必要的话一个对话框将会根据这个原因来重新校正它的尺寸。”So、可以看出，测量状态对谷歌官方而言也还算个测试性的功能，
            具体鄙人也没有找到很好的例证，如果大家谁找到了其具体的使用方法可以分享一下，这里我们还是就按照谷歌官方的建议依葫芦画瓢。
         */
        setMeasuredDimension(resolveSizeAndState(parentDesireWidth, widthMeasureSpec, childMeasureState),
                resolveSizeAndState(parentDesireHeight, heightMeasureSpec, childMeasureState << MEASURED_HEIGHT_STATE_SHIFT));
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (changed) {
            Log.d(TAG, String.format(" int l = %d, int t = %d, int  = %d, int b = %d", l, t, r, b));

            int count = getChildCount();
            int indexPoint = 1;//标识到了第几行或第几列
            int indexWidth = 0;//存储临时行的宽度
            int indexHeight = 0;//存储列的高度
            // 声明临时变量存储行/列临时宽高
            int tempHeight = 0, tempWidth = 0;

            CustomLayoutParams clp = null;
            View child = null;

            for (int i = 0; i < count; i++) {
                child = getChildAt(i);
                if (child.getVisibility() == View.GONE) {
                    continue;
                }
                clp = (CustomLayoutParams) child.getLayoutParams();
                if (mOrientation == ORIENTATION_HRIOZONTAL) {

                    if (count > mMaxColumn) {//多行
                        if (i < mMaxColumn * indexPoint) {
                            child.layout(
                                    getPaddingLeft() + indexWidth + clp.leftMargin,
                                    getPaddingTop() + indexHeight + clp.topMargin,
                                    getPaddingLeft() + indexWidth + clp.leftMargin + child.getMeasuredWidth(),
                                    getPaddingTop() + indexHeight + clp.topMargin + child.getMeasuredHeight()
                            );
                            tempHeight = Math.max(tempHeight, clp.topMargin + clp.bottomMargin + child.getMeasuredHeight());
                            indexWidth += child.getMeasuredWidth() + clp.leftMargin + clp.rightMargin;
                            if (i + 1 >= mMaxColumn * indexPoint) {
                                indexWidth = 0;
                                indexHeight += tempHeight;
                                indexPoint++;
                            }

                        }


                    } else {//单行

                        child.layout(
                                getPaddingLeft() + indexWidth + clp.leftMargin,
                                getPaddingTop() + clp.topMargin,
                                getPaddingLeft() + indexWidth + clp.leftMargin + child.getMeasuredWidth(),
                                getPaddingTop() + clp.topMargin + child.getMeasuredHeight()
                        );
                        indexWidth += child.getMeasuredWidth() + clp.leftMargin + clp.rightMargin;
                    }

                } else if (mOrientation == ORIENTATION_VERTICAL) {

                }
            }


        }
    }

    /**
     * 一致地返回false，其作用是告诉framework我们当前的布局不是一个滚动的布局
     * @return
     */
    @Override
    public boolean shouldDelayChildPressedState() {
        return false;
    }


    /*
     * 实现跟LayoutParams相关的方法
     */

    @Override
    protected LayoutParams generateDefaultLayoutParams() {
        return new CustomLayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    }

    @Override
    protected LayoutParams generateLayoutParams(LayoutParams p) {
        return new CustomLayoutParams(p);
    }

    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new CustomLayoutParams(getContext(), attrs);
    }

    @Override
    protected boolean checkLayoutParams(LayoutParams p) {
        return p instanceof CustomLayoutParams;
    }

    /**
     * 根据需求，自定义一个LayoutParams
     */
    public static class CustomLayoutParams extends MarginLayoutParams {

        public CustomLayoutParams(Context c, AttributeSet attrs) {
            super(c, attrs);
        }

        public CustomLayoutParams(int width, int height) {
            super(width, height);
        }

        public CustomLayoutParams(MarginLayoutParams source) {
            super(source);
        }

        public CustomLayoutParams(LayoutParams source) {
            super(source);
        }
    }
}

```


# 2 其他总结

1，在自定义控件时，如果控件事不需要滑动的，可以重写`shouldDelayChildPressedState`方法，告诉Framework控件不是不滚动的

2，Gravity的使用

    在自定义属性时：直接使用android:layout_gravity这样的name而无需定义类型值，
    这样则表示我们的属性使用的Android自带的标签，之后我们只需根据布局文件中layout_gravity属性的值调用Gravity类下的方法去计算对齐方式则可，
    Gravity类下的方法很好用，为什么这么说呢？因为其可以说是无关布局的，拿最简单的一个来说：
                    public static void apply(int gravity, int w, int h, Rect container, Rect outRect) 更多方法查看：http://www.programgo.com/article/42992498417/
                        gravity      所需放置的对象，由该类中的常量定义
                        w               对象的水平尺寸
                        h                对象的垂直尺寸
                        container          容器空间的框架，将用来放置指定对象，应该足够大，以包含对象的宽和高。
                        outRect    接收对象在其容器中的计算帧(computed frame)
