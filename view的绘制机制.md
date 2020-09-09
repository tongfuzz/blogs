##View的绘制机制

我们知道，ActivityThread在我们创建activity的时候会通过调用performLaunchActivity创建一个Activity对象，同时会调用activity对像的attach()方法,在attach()方法内部会为activity对象创建一个PhoneWindow对象，同时会为PhoneWindow设置一个WindowManager对象，用来管理我们的窗口对象，紧接着会调用mInstrumentation.callActivityOnCreate(）方法来调用activity的onCreate()方法，我们一般在onCreate()方法中会通过setContentView()来设置当前界面布局，setContentView()方法经过层层调用后会getWindow().setContentView(layoutResID)方法，即PhoneWindow对象的setContentView(layoutResID)方法，在此方法中我们会创建一个DecorView对象，作为我们页面的跟布局，同时将我们设置的布局文件作为content内容布局设置给DecorView中一个id为@android:id/content的framelayout作为子view,然后ActivityThread会接着调用handleResumeActivity(）方法来展示这个activity视图内容，在handleResumeActivity(）中会调用performResumeActivity(),在此方法中我们获取我们创建的WindowManager的addview方法将我们创建的DecorView对象添加到当前窗口中，在addView()方法中我们会调用WindowManagerGlobal的addView方法，在此方法中我们会创建一个ViewRootImpl对象，并通过调用ViewRootImpl对象的setView()方法将view对象设置给ViewRootImpl对象，ViewRootImpl是顶层视图，用来实现view和windowmanager的交互。

在调用了ViewRootImpl对象的setView()方法时，会调用requestlayout方法，发送一个runnable对像到消息队列中，然后通过run方法执行doTraversals()方法，然后执行performTraversals()方法，在此方法中会调用performMeasure(childWidthMeasureSpec, childHeightMeasureSpec)方法，来执行view的测量，紧接着会调用performLayout(lp, mWidth, mHeight);方法来进行摆放，然后会调用performDraw()方法来进行绘制，在performDraw方法中我们会调用draw(fullRedrawNeeded)方法来进行绘制，在draw(fullRedrawNeeded)方法中又会调用drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)方法来进行绘制，在此方法中会通过Surface对象来创建一个Canvas对象，然后调用 mView.draw(canvas)方法来对view进行绘制，至此DecorView绘制完毕，从上面的流程可以看出，绘制的过程主要在performMeasure(）performLayout(）performDraw()这三个方法中执行，在这三个方法中分别调用了view对象的measure(),layout(),draw()方法，下面我们来分析这三个方法的具体实现

###测量过程

####view.measure()方法分析

measure()方法位于View类中，是一个final修饰的方法，说明此方法不能被重写，

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
           ... 省略次要代码代码
       
               if (forceLayout || needsLayout) {
            //如果需要重绘
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
            		//如果没有从缓存中取出测量数据，那么我们自己测量            
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
             mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

可以看到在measure方法内部，会先从缓存中取测量的宽高值，如果取不到，就通过调用onMeasure(widthMeasureSpec, heightMeasureSpec)方法来自己测量，在上面的分析中我们知道，ViewRootImpl对象中的视图对象mView是一个DecorView对象，并且通过调用mView的measure()方法来进行了测量，那么在measure()方法中调用onMeasure(widthMeasureSpec, heightMeasureSpec)即是调用DecorView的onMeasure()方法，下面来看看DecorView的onMeasure()方法

```java
DecorView.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        ...忽略次要代码
        
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        
        ...忽略次要代码

        }
```
上述方法中进行了一系列的测量规则的计算，然后调用了父类的方法，DecorView继承自FrameLayout

```java
FrameLayout.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		  //获取子view的数量
        int count = getChildCount();
		  //测量模式的计算
		  //  父view      wrap_content   wrap_content    match_parent    match_parent  
		  //  子view      wrap_content   match_parent    wrap_content    match_parent 
		  //  测量规则         AT_MOST      AT_MOST          AT_MOST         EXECTLY
		  //由于FrameLayout的测量模式是MeasureSpec.EXACTLY 所以返回false，如果framlayout设置的不是MATCH_PARENT
		  //则需要测量设置为match_parent的子view
        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();
		 //最大宽度和高度
        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;
		 //循环遍历子view，并测量他们的宽高，获取最大的宽高值
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
            	   //测量子view的宽高，注意，这里子view的宽高并不包含父view的padding和子view的margin值
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                //获取子view的布局参数
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //获取子view的测量宽度以及左右margin值，与当前记录的最大值比较
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                //获取子view的测量高度以及上下margin值，与当前记录的最大值比较
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
             
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        //将测量的最大宽高分别加上当前view的padding值，从新作为最大宽高值
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        // 判断最大宽高值和建议的最小宽高值中取最大值
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }
		 //设置宽高值
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }
                
                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }

                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }
```

上面framelayout的onMeasure方法中，我们通过循环遍历子view，测量子view，并获取到所有子view中最高和最宽值，设置给当前framelayout，至此我们的最外层view已经测量完毕了，并得到了宽高值和测量模式，在最开始的分析中我们知道，在最外层的framelayout中有一个id为@android:id/content的framelayout,这个framelayout即为我们的内容布局，我们通过setContentView设置的布局文件就是通过mLayoutInflater.inflate(layoutResID, mContentParent)方法作为子view添加到了此内容布局中，在我们上面的循环遍历子view的过程中，自然会遍历到此内容布局framelayout，它的测量过程与上面的一样（因为都是framelayout）我们就不再分析，下面我们来看内容布局中的子view即我们设置的布局文件的测量，假设我们设置的最外层是一个linearlayout，通过内容布局的onMeasure方法中的measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0）我们来进行子view的测量,来看看此方法

```java
ViewGroup.java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        //获取子view的布局参数
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
		 //获取子view的测量规则
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
		  //调用子view的measure（)方法开始子view的测量
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
从上面的方法中看出，我们先获取子view的布局参数，然后生成子view的测量规则，最后才调用measure方法进行测量，测量规则的生成很重要，我们先来看看

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        //获取父view的测量值和测量规则
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        //specSize为父view测量最大值，padding为父view的padding以及子view的margin值
        //通过二者相减求出子view所能得到的最大的宽高值
        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        //父view有确定宽高，
        case MeasureSpec.EXACTLY:
            //childDimension为固定值时，说明我们给子view的layout_width或layout_height设置了具体的宽高
            if (childDimension >= 0) {
                //设置子view的值为固定值，测量规格为MeasureSpec.EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //如果我们设置成了LayoutParams.MATCH_PARENT，设置子view的值为最大值，                           
                //测量规格为MeasureSpec.EXACTLY          
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //如果设置成了LayoutParams.WRAP_CONTENT，设置子view的测量值所能达到的最大值为size
                //设置测量规格为MeasureSpec.AT_MOST，注意resultSize = size说明子view所能达到的最大值为size
                //并不是说就是size
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        //如果父view的测量规格为AT_MOST
        case MeasureSpec.AT_MOST:
            //childDimension为固定值时，说明我们给子view的layout_width或layout_height设置了具体的宽高
            if (childDimension >= 0) {
                ////设置子view的值为固定值，测量规格为MeasureSpec.EXACTLY
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                //子view的测量值设置为父view的测量值，由于父view并没有确定宽高，所以，子view测量值也不确定
                //只能设置不超过最大值size
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //子view的测量值设置为父view的测量值，由于父view并没有确定宽高，所以，子view测量值也不确定
                //只能设置不超过最大值size
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // 没有父view没有指定宽高
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //创建测量规格
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
在上面我们会根据父view的测量规格和子view的布局参数来生成子view的测量规格，在获取到子view的测量规格后，我们通过child.measure(childWidthMeasureSpec, childHeightMeasureSpec)开始了子view的测量，在上面我们已经假设我们的布局文件最外层时LinearLayout,那么我们就接着来看LinearLayout的measure方法,查找发现linearlayout并没有measure（）方法，此方法是View类中的方法，而在measure()方法中主要测量任务交给了onMeasure(widthMeasureSpec, heightMeasureSpec)方法，下面我们来看看LinearLayout的onMeasure(widthMeasureSpec, heightMeasureSpec)方法

```java
	LinearLayout.java
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
        	  //如果LinearLayout的方向是垂直的，调用垂直测量方法
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            //如果LinearLayout的方向是水平的，调用水平测量方法
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
```
下面我们先来看垂直测量方法

```java
LinearLayout.java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
        //总高度
        mTotalLength = 0;
        //最大宽度
        int maxWidth = 0;
        int childState = 0;
        int alternativeMaxWidth = 0;
        int weightedMaxWidth = 0;
        boolean allFillParent = true;
        //总权重
        float totalWeight = 0;
		 //获取垂直方向子view的个数
        final int count = getVirtualChildCount();
		 //获取当前view的测量模式
        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        boolean matchWidth = false;
        boolean skippedMeasure = false;
	    
        final int baselineChildIndex = mBaselineAlignedChildIndex;
        final boolean useLargestChild = mUseLargestChild;

        int largestChildHeight = Integer.MIN_VALUE;
        int consumedExcessSpace = 0;

		  //没有跳过测量的子view的个数
        int nonSkippedChildCount = 0;

        //循环遍历当前子view
        for (int i = 0; i < count; ++i) {
            //获取垂直方向子view
            final View child = getVirtualChildAt(i);
            //如果子view为null，继续下一个view
            if (child == null) {
                mTotalLength += measureNullChild(i);
                continue;
            }
			  //如果子view不可见，跳过，继续下一个view
            if (child.getVisibility() == View.GONE) {
               i += getChildrenSkipCount(child, i);
               continue;
            }
			  //没有跳过测量的子view的个数加1
            nonSkippedChildCount++;
            if (hasDividerBeforeChildAt(i)) {
                //如果在当前view的前面有分割线，将分割线计入总高度
                mTotalLength += mDividerHeight;
            }
			  //获取子view的布局参数
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
			  //将子view的权重值计入到总权重值中，如果没有设置权重，默认为0
            totalWeight += lp.weight;
            //如果高度为0，并且权重大于0，说明需要使用权重
            final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;
            //如果高度的测量模式为EXACTLY，并且需要使用权重，这时我们需要先跳过这个子view的测量，
            if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
                //虽然我们跳过了这个子view的测量，但是如果这个子view设置的有topMargin和bottomMargin，
                //我们仍然需要将其值计入到总高度上
                mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
                //标记为跳过测量
                skippedMeasure = true;
            } else {
                //如果高度的测量模式不是EXACTLY，并且需要使用权重
                if (useExcessSpace) {
                   
                    //说明测量模式为UNSPECIFIED或者AT_MOST，这时我们将此子view的布局参数高						//度设置为WRAP_CONTENT，以便我们后续为它设置合适高度
                    lp.height = LayoutParams.WRAP_CONTENT;
                }

                //如果权重总值等于0，说明还没有子view使用权重，那么usedHeight已经使用的值即为总高度，
                //如果权重总值不等于0，说明有子view使用权重，那么usedHeight已经使用的值0
                final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
                //根据已经使用的高度的值，和测量规格来测量子view，当子view使用了权重时，我们
                //先允许它使用所有的有效空间，后面可以在收缩
                measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                        heightMeasureSpec, usedHeight);
				   //获取子view的测量高度
                final int childHeight = child.getMeasuredHeight();
                //如果使用权重
                if (useExcessSpace) {
                    //前面使用权重，我们将高度暂时设置为了LayoutParams.WRAP_CONTENT以便测量高度，
                    //在测量完成后，我们将高度值还原，并记录使用权重的view的测量的高度最大值
                    lp.height = 0;
                    //记录设置了权重比的view的测量的高度的最大值为多少
                    consumedExcessSpace += childHeight;
                }

                final int totalLength = mTotalLength;
                //求出linearlayout的总高度
                mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                       lp.bottomMargin + getNextLocationOffset(child));
				   //计算高度最高的view
                if (useLargestChild) {
                    largestChildHeight = Math.max(childHeight, largestChildHeight);
                }
            }

          
            if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
               mBaselineChildTop = mTotalLength;
            }

          
            if (i < baselineChildIndex && lp.weight > 0) {
                throw new RuntimeException("A child of LinearLayout with index "
                        + "less than mBaselineAlignedChildIndex has weight > 0, which "
                        + "won't work.  Either remove the weight, or don't set "
                        + "mBaselineAlignedChildIndex.");
            }

            boolean matchWidthLocally = false;
            //如果高度测量模式为EXACTLY，并且子view高度设置为MATCH_PARENT
            if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
                //标记有子view设置了宽度为父view的宽度
                matchWidth = true;
                matchWidthLocally = true;
            }
			  //计算子view的margin值
            final int margin = lp.leftMargin + lp.rightMargin;
            //计算子view的宽度
            final int measuredWidth = child.getMeasuredWidth() + margin;
            //与最大宽度比较，取最大值
            maxWidth = Math.max(maxWidth, measuredWidth);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            
            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
            //如果子view设置了权重
            if (lp.weight > 0) {
               
                //如果我们使用权重，求权重最大宽度
                weightedMaxWidth = Math.max(weightedMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            } else {
                //
                alternativeMaxWidth = Math.max(alternativeMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            }

            i += getChildrenSkipCount(child, i);
        }
        //如果没有跳过测量的view的数量大于0，并且有分割符
        if (nonSkippedChildCount > 0 && hasDividerBeforeChildAt(count)) {
            //总高度添加分割符高度
            mTotalLength += mDividerHeight;
        }
        //
        if (useLargestChild &&
                (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
            mTotalLength = 0;

            for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);
                if (child == null) {
                    mTotalLength += measureNullChild(i);
                    continue;
                }

                if (child.getVisibility() == GONE) {
                    i += getChildrenSkipCount(child, i);
                    continue;
                }

                final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                        child.getLayoutParams();
                // Account for negative margins
                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                        lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
            }
        }

        //添加padding到总高度中
        mTotalLength += mPaddingTop + mPaddingBottom;

        int heightSize = mTotalLength;
        //计算高度值和建议最小高度值的最大值
        heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

        //计算大小和状态，如果测量模式为EXACTLY，高度即为测量值，如果为AT_MOST，并且测量值小于计算值，取测量值
        //并且测量状态为MEASURED_STATE_TOO_SMALL，否则，取计算值
        int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
        heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
        
        
        //计算剩余空间，注意此处有可能为负值
        int remainingExcess = heightSize - mTotalLength
                + (mAllowInconsistentMeasurement ? 0 : consumedExcessSpace);
        if (skippedMeasure || remainingExcess != 0 && totalWeight > 0.0f) {
            //如果使用了权重,前面我们已经按照权重的计算了所需要的空间，然后计算出了剩余空间
            //接下来我们要根据剩余空间从新测量
            
            //权重值
            float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;
            //总高度置为0
            mTotalLength = 0;
            //循环遍历view
            for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }
                //获取view的布局参数
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //获取权重
                final float childWeight = lp.weight;
             
                if (childWeight > 0) {
                    //如果权重大于0，计算相应的权重所占高度为多少
                    final int share = (int) (childWeight * remainingExcess / remainingWeightSum);
                    //减去这些空间
                    remainingExcess -= share;
                    //减去这些权重
                    remainingWeightSum -= childWeight;
                    //
                    final int childHeight;
                    //如果需要根据最大子view的高度来
                    if (mUseLargestChild && heightMode != MeasureSpec.EXACTLY) {
                        childHeight = largestChildHeight;
                    } else if (lp.height == 0 && (!mAllowInconsistentMeasurement
                            || heightMode == MeasureSpec.EXACTLY)) {
                        //使用计算出来的view作为高度
                        childHeight = share;
                    } else {
                        //使用计算出来的高度和view本身的高度作为view的高度
                        childHeight = child.getMeasuredHeight() + share;
                    }
                    //在此计算测量规则，在此测量子view，如果childHeight为负值，则设置高度为0
                    final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            Math.max(0, childHeight), MeasureSpec.EXACTLY);
                    final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                            lp.width);
                    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                    // Child may now not fit in vertical dimension.
                    childState = combineMeasuredStates(childState, child.getMeasuredState()
                            & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
                }

                final int margin =  lp.leftMargin + lp.rightMargin;
                final int measuredWidth = child.getMeasuredWidth() + margin;
                maxWidth = Math.max(maxWidth, measuredWidth);
                
                boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                        lp.width == LayoutParams.MATCH_PARENT;

                alternativeMaxWidth = Math.max(alternativeMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);

                allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

                final int totalLength = mTotalLength;
                //计算总高度
                mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                        lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
            }

            // 添加padding
            mTotalLength += mPaddingTop + mPaddingBottom;
            // TODO: Should we recompute the heightSpec based on the new total length?
        } else {
            //计算高度
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                                           weightedMaxWidth);


            // We have no limit, so make all weighted views as tall as the largest child.
            // Children will have already been measured once.
            if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
                for (int i = 0; i < count; i++) {
                    final View child = getVirtualChildAt(i);
                    if (child == null || child.getVisibility() == View.GONE) {
                        continue;
                    }

                    final LinearLayout.LayoutParams lp =
                            (LinearLayout.LayoutParams) child.getLayoutParams();

                    float childExtra = lp.weight;
                    if (childExtra > 0) {
                        child.measure(
                                MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                        MeasureSpec.EXACTLY),
                                MeasureSpec.makeMeasureSpec(largestChildHeight,
                                        MeasureSpec.EXACTLY));
                    }
                }
            }
        }

        if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
            maxWidth = alternativeMaxWidth;
        }

        maxWidth += mPaddingLeft + mPaddingRight;

        // Check against our minimum width
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
        //设置测量值
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                heightSizeAndState);

        if (matchWidth) {
            forceUniformWidth(count, heightMeasureSpec);
        }
    }
```
上面的测量过程可以分为两个情况，一种是用户设置了权重比，一种是没有设置，在用户设置了权重比的情况下，我们先测量没有设置权重比的view，然后通过将设置了权重比的view的height设置为wrap_content来测量他们的总高度，然后再根据测量规则求出的最大的高度，然后和我们测量的高度相减，求出剩余的空间，然后再根据权重比计算出设置了权重的view的高度，最后再通setMeasuredDimension(）设置高度，至此LinearLayout的测量过程已经完成。

至于水平方向的测量与垂直方向的测量过程原理相同，只是方向不同罢了


###摆放(Layout)过程
在文章开头的分析中我们知道，在ViewRootImpl中调用完performMeasure(childWidthMeasureSpec, childHeightMeasureSpec)后我们会调用performLayout(lp, mWidth, mHeight)来进行view的布局，来看看这个方法

```java
ViewRootImpl.java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        //ViewRootImpl类中的mView即是我们创建的DecorView
        final View host = mView;
        if (host == null) {
            return;
        }
        ...忽略次要代码
        //调用DecorView的layout方法，四个参数分别为左上右下四边距离父view的位置，host.getMeasuredWidth()
        //和host.getMeasuredHeight()为屏幕的宽高
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        ...忽略次要代码  
    }
```
在上面的方法中主要通过DecorView的layout方法来完成布局，DecorView继承自FrameLayout，也就是一个View类的子类，下面我们来看看View类中的layout方法

```java
   View.java
   public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
      
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        //
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            //如果我们请求layout，调用onLayout方法
            onLayout(changed, l, t, r, b);

            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }

            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

        if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
            mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
            notifyEnterOrExitForAutoFillIfNeeded(true);
        }
    } 
```
在view中的layout的方法，主要是通过onLayout方法来进行摆放子view的，而各个子view也是通过重写此方法来完成自己的摆放逻辑的，由于DecorView是继承自FrameLayout，来看看FrameLayout方法的onLayout方法

```java
FrameLayout.java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }
```
调用了自身的layoutChildren（）方法

```java
void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        //获取子view的数量
        final int count = getChildCount();
        //获取当前view的上下左右位置
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
        //循环子view
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            //如果view没有设置为GONE
            if (child.getVisibility() != GONE) {
                //获取子view的布局参数
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
					//获取子view的宽高
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
                //用来记录子view的左边界和上边界
                int childLeft;
                int childTop;
					//获取子view的layout_gravity属性
                int gravity = lp.gravity;
                if (gravity == -1) {
                    //如果没有设置gravity属性
                    gravity = DEFAULT_CHILD_GRAVITY;
                }
                //获取此view的布局方向,framelayout有两个布局方向，从左向右或者从右向左
                final int layoutDirection = getLayoutDirection();
                //根据framelayout中的layoutDirection和子view的layout_gravity属性来计算位置
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                //计算子view的竖直方向Gravity
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
				   //计算子view的水品方向的Gravity
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        //如果是水平居中
                        //计算子view的左边位置，应该为父view的左边距离加上父view的宽度减去子view
                        //的值的1/2，再加上左边的margin距离，减去右边的margin距离
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        //如果子view的layout_gravity设置为RIGHT，说明要放在右边
                        if (!forceLeftGravity) {
                            //计算子view的左边位置，应该为父view的右边位置减去子view的宽度，再减
                            //去右侧margin值
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        //如果子view的layout_gravity设置为LEFT或者没有设置，计算子view的左边位置，为父
                        //view的左边位置加上左侧margin值
                        childLeft = parentLeft + lp.leftMargin;
                }

                switch (verticalGravity) {
                    //计算子view顶部位置
                    case Gravity.TOP:
                        //如果子view的layout_gravity设置为TOP，则子view的顶部的位置为父view的
                        //顶部位置加上子view的顶部margin值
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        //如果子view的layout_gravity设置为CENTER_VERTICAL
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        //如果子view的layout_gravity设置为BOTTOM，子view的顶部为父view的底部减去子
                        //view的高度，以及子view的地步margin
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }
                自此，子view的左侧与顶部位置已经确定，接下来测量子view的子view
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```
在上面的方法中可以看出，framelayout的子view的摆放，首先是获取当前view的左侧和顶部的位置，然后根据子view的布局方式以及左侧和顶部margin来摆放子view，然后再调用子view的layout方法来进行子view的子view的摆放，我们设置的layout的布局文件就是framelayout的子view，假设我们布局文件的最外层设置的是linearlayout，我们来看看linearlayout的layout方法，由于linearlayout和framelayout方法都是view的子view，所以layout方法一样，我们直接来看linearlayout中的onlayout方法

####Linearlayout的layout过程

```java
LinearLayout.java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
        //跟测量过程一样，layout同样是根据布局方向来进行不同的操作
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```
先来看看layoutVertical(l, t, r, b)

```java
LinearLayout.java
void layoutVertical(int left, int top, int right, int bottom) {
        //linearlayout的paddingLeft的值
        final int paddingLeft = mPaddingLeft;
        //子view的顶部和左部位置
        int childTop;
        int childLeft;

        //LinearLayout所能提供的最大宽度
        final int width = right - left;
        
        //子view相对linearlayout的右侧右侧边界
        int childRight = width - mPaddingRight;

        //去除linearlayout的左右padding值后能为子view提供的最大空间
        int childSpace = width - paddingLeft - mPaddingRight;
        //获取子view的数量
        final int count = getVirtualChildCount();
        
        //linearlayout的竖直gravity的值
        final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        //linearlayout的水平gravity的值
        final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
       
        switch (majorGravity) {
           case Gravity.BOTTOM:
               //如果gravity设置的值为Gravity.BOTTOM，那么子view的顶部位置为linearlayout的paddingtop值
               //再加上加上bottom - top的值，此处即为linerlayout的实际底部位置，然后减去子view的总高度
               //得到子view的顶部位置
               childTop = mPaddingTop + bottom - top - mTotalLength;
               break;

               // mTotalLength contains the padding already
           case Gravity.CENTER_VERTICAL:
               //如果gravity设置的值为Gravity.CENTER_VERTICAL，那么子view的顶部的位置为linearlayout的
               //高度减去子view的总高度的到的空闲空间的1/2，然后再加上顶部padding值
               childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
               break;

           case Gravity.TOP:
           default:
               //如果没有设置gravity值或者设置为 Gravity.TOP，那么子view的顶部的位置即为父view的顶部位置
               //加上padding值
               childTop = mPaddingTop;
               break;
        }

        for (int i = 0; i < count; i++) {
            //循环遍历子view
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                //如果子view的状态不为GONE
                
                //获取子view的宽度和高度
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();
					//获取子view的布局参数
                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                //获取子view的layout_gravity属性值
                int gravity = lp.gravity;
                //如果子view没有设置layout_gravity属性值，此处说明了，如果子view设置了layout_gravity
                //那么以子view的的layout_gravity为主
                if (gravity < 0) {
                    //取父view的gravity值
                    gravity = minorGravity;
                }
                //获取linearlayout的布局方向
                final int layoutDirection = getLayoutDirection();
                //根据子view的layout_gravity和linearlayout的布局方向来计算子view的绝对layout_gravity值
                //此值在left right center_horiziontal 三者之间
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
               
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        //如果是Gravity.CENTER_HORIZONTAL，那么子view的左侧位置为linearlayout
                        //的paddingLeft值，加上子view自身的leftMargin值，再加上空闲空间的1/2，然后减去
                        //自身的rightMargin的值
                        childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                                + lp.leftMargin - lp.rightMargin;
                        break;

                    case Gravity.RIGHT:
                        //如果子view的layout_gravit值为Gravity.RIGHT，左侧位置为最大宽度，减去子view的
                        //宽度，再减去子view的rightMargin
                        childLeft = childRight - childWidth - lp.rightMargin;
                        break;

                    case Gravity.LEFT:
                    default:
                        //如果子view没有设置gravity值，那么默认放在左边，左侧位置为linearlayout的paddingleft值
                        //加上leftMargin
                        childLeft = paddingLeft + lp.leftMargin;
                        break;
                }
					//如果有divider，childTop加上divider的高度
                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }
                //如果子view设置的有topmargin，childTop加上topmargin的值
                childTop += lp.topMargin;
                //调用子view的layout方法，并将获得的left，top，right，bottom值传入，这时已经确定了上下左右的位置
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                //为下一个子view设置childTop值，下一个子view的childTop值应该为当前子view的childTop值加上当前子view的高度
                //以及当前子view的bottomMargin，以及下一个子view的偏移量
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }

```
从上面的过程可以看出，linearlayout先根据它在measure过程的到的子view的总高度，和自身的padding值来计算开始的childTop的值，然后循环遍历view，根据view的margin值，和view的高度来摆放view的，如过子view是viewgroup的话，会接着进行摆放的，不同的viewgroup摆放规则不一样，自此view的layout过程已经完毕

###Draw过程
上面的layout过程完成以后，ViewRootImpl会接着调用performDraw()方法

```java
 ViewRootImpl.java
 private void performDraw() {
        ...忽略次要代码
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        ...忽略次要代码
         
    }
```
调用了draw方法

```java
 ViewRootImpl.java
 private void draw(boolean fullRedrawNeeded) {
        ...忽略次要代码
                 //如果我们使用硬件加速
        			 mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        			//使用软件方式进行绘制
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                    return;
                }
        ...忽略次要代码  
      }
```
在上面的方法中，先创建了一个surface对象，然后调用了drawSoftware方法

```java
ViewRootImpl.java
 private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {
            
        final Canvas canvas;
       
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;
			  //根据surface对象获取一个Canvas对象
            canvas = mSurface.lockCanvas(dirty);
            canvas.setDensity(mDensity);
      
			  ...省略次要代码
			  
            dirty.setEmpty();
            mIsAnimating = false;
            mView.mPrivateFlags |= View.PFLAG_DRAWN;

                canvas.translate(-xoff, -yoff);
                if (mTranslator != null) {
                    mTranslator.translateCanvas(canvas);
                }
                canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                attachInfo.mSetIgnoreDirtyState = false;
					//调用mView，即DecorView的draw方法
                mView.draw(canvas);

                drawAccessibilityFocusedDrawableIfNeeded(canvas);
                
                   return true;
    }
```
在上面的方法中，我们根据创建的surface对象获取了一个Canvas，然后调用了mView.draw(canvas)方法开始了view的绘制，来看看DecorView的draw()方法

```java
DecorView.java
 @Override
    public void draw(Canvas canvas) {
        //调用了父view的draw(canvas)方法
        super.draw(canvas);
        //绘制menu背景
        if (mMenuBackground != null) {
            mMenuBackground.draw(canvas);
        }
    }
```
调用了父view的draw方法

```java
View.java
public void draw(Canvas canvas) {
            ...忽略次要代码
            // 绘制自己
            if (!dirtyOpaque) onDraw(canvas);

            //绘制子view
            dispatchDraw(canvas);
			  
            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
       }

```
在View类的draw方法中，首先调用了onDraw(canvas)方法，我们来进行自身的绘制，在LinearLayout,FrameLayout中，一般此方法为空实现，因为它本身是一个用来放view的ViewGroup,一般我们对它并没有什么东西需要绘制，一般我们要绘制的是子view，接下来，我们调用了dispatchDraw(canvas)方法，分发绘制过程,dispatchDraw(canvas)方法是ViewGroup中的方法

```java
protected void dispatchDraw(Canvas canvas) {
        boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        int flags = mGroupFlags;
        int clipSaveCount = 0;
       
        final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
        
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                        //调用drawChild方法绘制子view
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }

            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
       }
```

在ViewGroup的dispatchDraw(canvas)方法中，我们调用了drawChild(canvas, transientChild, drawingTime)方法来绘制view