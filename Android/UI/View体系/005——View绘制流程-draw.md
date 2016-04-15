# 1 Draw流程
绘制是View树遍历流程的最后一个，用来绘制View，前面说的测量和布局只是确定View的大小和位置，如果不对view进行绘制，那么界面上依然不会有任何图形显示出来，draw也是从ViewRoot中的performTraversals发起的。这里会调用view的draw方法，但是并不是每个View都需要执行绘制，在执行绘制的过程中，只会重绘需要绘制的View


# 2 draw源码分析

     public void draw(Canvas canvas) {
           ......
           //通过内部标识，判断View的行为
            final int privateFlags = mPrivateFlags;
            final boolean dirtyOpaque = (privateFlags & DIRTY_MASK) == DIRTY_OPAQUE &&
                    (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
            mPrivateFlags = (privateFlags & ~DIRTY_MASK) | DRAWN;

            /*
             *draw的步骤
             *
             *      1. 画背景
             *      2. 如果需要, 为显示渐变框做一些准备操作
             *      3. 画内容(onDraw)
             *      4. 画子view
             *      5. 如果需要, 画一些渐变效果
             *      6. 画装饰内容，如滚动条
             */

            // Step 1, draw the background, if needed
            int saveCount;

            if (!dirtyOpaque) {
                final Drawable background = mBGDrawable;
                if (background != null) {
                    final int scrollX = mScrollX;
                    final int scrollY = mScrollY;

                    if (mBackgroundSizeChanged) {
                        background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
                        mBackgroundSizeChanged = false;
                    }

                    if ((scrollX | scrollY) == 0) {
                        background.draw(canvas);
                    } else {
                        canvas.translate(scrollX, scrollY);
                        background.draw(canvas);
                        canvas.translate(-scrollX, -scrollY);
                    }
                }
            }

            // skip step 2 & 5 if possible (common case)
            final int viewFlags = mViewFlags;
            boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
            boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    //如果条件不成立，跳过2-5步
            if (!verticalEdges && !horizontalEdges) {
                // Step 3,画内容
                if (!dirtyOpaque) onDraw(canvas);

                // Step 4,画孩子
                dispatchDraw(canvas);

                // Step 6, 画装饰（滚动条）
                onDrawScrollBars(canvas);

                // we're done...
                return;
            }

           ......

            // Step 6, draw decorations (scrollbars)
            onDrawScrollBars(canvas);
        }


**dispatchDraw&drawChild**

    protected void dispatchDraw(Canvas canvas) {
    ......

     for (int i = 0; i < count; i++) {
                    final View child = children[i];
                    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                        more |= drawChild(canvas, child, drawingTime);
                    }
                }


    ......
    }


    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
            return child.draw(canvas, this, drawingTime);
        }


可以看到draw的过程分为：
1. 如果设置了，画背景
2. 如果需要, 为显示渐变框做一些准备操作
3. 调用onDraw画内容
4. 调用dispatchDraw画子view
5. 如果需要, 画渐变框
6. 画装饰内容，如滚动条


- onDraw是每个view需要实现的，否则View默认只能显示背景，而实现onDraw就是为了画出View的内容，而ViewGroup一般不需要实现onDraw,因为它仅仅是作为View的容器没有需要绘制东西，
- dispatchDraw用来遍历ViewGrop的所有子view，执行draw方法
- draw不是fianl方法，我们可以重载此方法，完全自定义draw过程，但是这不是很明智的选择


# 3 onDraw中如何绘制

在系统源码中onDraw是个空实现方法，仅仅提供了一个Canvas画板，到底如何来画View的内容呢？

如果需要熟练的绘制出各种效果的View，我们需要掌握很多知识：

- Canvas的是使用 绘制-变化-图层操作等等
- Paint 画笔
- Path 路径
- Bitmap Canvas是画布，但是我们需要画纸，Bitmap就是画纸
- ColorMatrix和Matrix的熟练运用

总之绘制需要掌握很多的知识。
