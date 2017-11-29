---
title: Android中SwipeBack右滑返回（手势返回）的实现
date: 2015/12/09 12:00:00
tags:
- Android 
- 自定义控件 
- swipeback
- ui
categories: 
- Android一周自定义控件

---

现在有很多App支持右滑返回，比如知乎，效果比较赞。

![这里写图片描述](http://img.blog.csdn.net/20151209115408972)

于是自己对Activity和Fragment进行了继承，派生出SwipeBackActivity和SwipeBackFragment，用于对这种效果的实现，也就是只要继承这两个类就可以了。

__效果如下__

- Activity
![这里写图片描述](http://img.blog.csdn.net/20151209115440495)

- Fragment
![这里写图片描述](http://img.blog.csdn.net/20151209115536875)


Frgament的效果实现比Activity稍微简单，因为Activity要考虑到dectorView。

支持滑动的控件__SwipeLayout__,核心思路就是把原有的控件添加到支持滑动的控件中，__SwipeLayout__要注意计算手势速度，源码如下：

``` java
package com.ui.jerry.swipebackdemo;

import android.content.Context;
import android.util.AttributeSet;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.MotionEvent;
import android.view.VelocityTracker;
import android.view.View;
import android.view.ViewConfiguration;
import android.view.ViewGroup;
import android.widget.LinearLayout;
import android.widget.Scroller;
import android.widget.Toast;

/**
 * @author jerry
 * @version 4.6
 * @date 2015-10-27
 */
public class SwipeLayout extends LinearLayout {
    public static final String TAG = "SwipeLayout";

    private View mEmptyView;
    private View mContentView;

    private int mLeftEdge;
    private int mWidth;
    private int mMaxScrollX;

    private Scroller mScroller;
    private VelocityTracker mVelocityTracker = null;
    private int mMaxFlingVelocity;
    private int mLastX;

    ViewGroup.LayoutParams childParams = new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);

    private Context mContext;
    public static final int DURATION = 1500;   //满屏滑动时间
    public static final int OPEN_ANIM_DURATION = 1000;
    public static int SNAP_VELOCITY = 600;  //最小的滑动速率

    private OnFinishListener mOnFinishListener;

    public SwipeLayout(Context context) {
        this(context, null);
    }

    public SwipeLayout(Context context, AttributeSet attrs) {
        super(context, attrs);

        mContext = context;
        init();
    }

    public void setOnFinishListener(OnFinishListener mOnFinishListener) {
        this.mOnFinishListener = mOnFinishListener;
    }

    void init() {
        mScroller = new Scroller(mContext);
        mMaxFlingVelocity = ViewConfiguration.get(mContext).getScaledMaximumFlingVelocity();

        mWidth = DisplayUtils.getScreenWidth(mContext) * 2;
        mMaxScrollX = mWidth / 2;
        mLeftEdge = mMaxScrollX - mMaxScrollX / 3;

        setOrientation(LinearLayout.HORIZONTAL);

        childParams.width = DisplayUtils.getScreenWidth(mContext);

        mEmptyView = LayoutInflater.from(mContext).inflate(R.layout.view_translate, null);

        addView(mEmptyView, childParams);
    }

    public void setContentView(View contentView) {
        if (mContentView != null) {
            removeView(mContentView);
        }
        mContentView = contentView;
        addView(contentView, childParams);

        postDelayed(new Runnable() {
            @Override
            public void run() {
                openActivityAnimation();
            }
        }, 200);
    }

    /**
     * 获取速度追踪器
     *
     * @return
     */
    private VelocityTracker getVelocityTracker() {
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        return mVelocityTracker;
    }

    /**
     * 回收速度追踪器
     */
    private void recycleVelocityTracker() {
        if (mVelocityTracker != null) {
            mVelocityTracker.clear();
            mVelocityTracker.recycle();
            mVelocityTracker = null;
        }
    }


    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        //1.获取速度追踪器
        getVelocityTracker();
        //2.将当前事件纳入到追踪器中
        mVelocityTracker.addMovement(ev);

        int pointId = -1;

        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                //如果屏幕的动画还没结束，你就按下了，我们就结束上一次动画，即开始这次新ACTION_DOWN的动画
//                clearScrollHis();
                mLastX = (int) ev.getX();
                pointId = ev.getPointerId(0);
                break;
            case MotionEvent.ACTION_MOVE:
                int nextScrollX = (int) (mLastX - ev.getX() + getScrollX());

                if (scrollTo(nextScrollX)) {
                    mLastX = (int) ev.getX();
                }
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                //3.计算当前速度
                mVelocityTracker.computeCurrentVelocity(1000, mMaxFlingVelocity);
                //获取x y方向上的速度
                float vX = mVelocityTracker.getXVelocity(pointId);

                Log.i(TAG, "mVelocityX:" + vX);

                //大于某个速率 直接滑动
                if (vX > SNAP_VELOCITY) {
                    scrollToLeft();
                } else if (vX < -SNAP_VELOCITY) {
                    scrollToRight();
                } else {
                    snapToDestation();
                }


                //4.回收速度追踪器
                recycleVelocityTracker();
                break;
        }
        return true;
    }

    private void openActivityAnimation() {
        clearScrollHis();
        mScroller.startScroll(getScrollX(), 0, mMaxScrollX - getScrollX(), 0, OPEN_ANIM_DURATION);
        invalidate();//这里必须调用invalidate()才能保证computeScroll()会被调用，否则不一定会刷新界面，看不到滚动效果
    }

    public void closeActivityAnimation() {
        clearScrollHis();
        mScroller.startScroll(getScrollX(), 0, -getScrollX(), 0, OPEN_ANIM_DURATION);
        invalidate();//这里必须调用invalidate()才能保证computeScroll()会被调用，否则不一定会刷新界面，看不到滚动效果
    }

    private void clearScrollHis() {
        if (mScroller != null) {
            if (!mScroller.isFinished()) {
                mScroller.abortAnimation();
            }
        }
    }

    /**
     * 根据现在的滚动位置判断
     */
    private void snapToDestation() {
        int scrollX = getScrollX();
        if (scrollX > 0 && scrollX <= mLeftEdge) {
            smoothScrollTo(0);
        } else if (scrollX > mLeftEdge) {
            smoothScrollTo(mMaxScrollX);
        }
    }

    /**
     * 直接滚动
     *
     * @param x
     * @return
     */
    public boolean scrollTo(int x) {
        if (x < 0) {
            scrollTo(0, 0);
        } else if (x > mMaxScrollX) {
            scrollTo(mMaxScrollX, 0);
        } else {
            scrollTo(x, 0);
        }
        return true;
    }

    public void scrollToRight() {
        smoothScrollTo(mMaxScrollX);
    }

    public void scrollToLeft() {
        smoothScrollTo(0);
    }

    @Override
    protected void onScrollChanged(int l, int t, int oldl, int oldt) {
        super.onScrollChanged(l, t, oldl, oldt);

        Log.d(TAG, "left:" + l);

        if (l == 0) {
            Log.d(TAG, "OnFinish");

            Toast.makeText(mContext, "Finished", Toast.LENGTH_SHORT).show();

            if(mOnFinishListener!=null){
                mOnFinishListener.onFinish();
            }
        }
    }

    public void smoothScrollTo(int fx) {
        if (fx < 0) {
            smoothScrollTo(0, 0);
        } else if (fx > mMaxScrollX) {
            smoothScrollTo(mMaxScrollX, 0);
        } else {
            smoothScrollTo(fx, 0);
        }
    }

    //    //调用此方法滚动到目标位置
    public void smoothScrollTo(int fx, int fy) {
        int dx = fx - getScrollX();
        int dy = fy - getScrollY();
        smoothScrollBy(dx, dy);
    }

    //调用此方法设置滚动的相对偏移
    public void smoothScrollBy(int dx, int dy) {

        //设置mScroller的滚动偏移量
        mScroller.startScroll(getScrollX(), 0, dx, dy, Math.abs(dx * DURATION / mMaxScrollX));
        invalidate();//这里必须调用invalidate()才能保证computeScroll()会被调用，否则不一定会刷新界面，看不到滚动效果
    }

    @Override
    public void computeScroll() {

        //先判断mScroller滚动是否完成
        if (mScroller.computeScrollOffset()) {

            //这里调用View的scrollTo()完成实际的滚动
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());

            //必须调用该方法，否则不一定能看到滚动效果
            postInvalidate();
        }
        super.computeScroll();
    }

    /**
     * fragment或者activity 结束的接口
     */
    public interface OnFinishListener{
        void onFinish();
    }
}

```


[__下载地址__](http://download.csdn.net/detail/huweigoodboy/9338137)

