# RippleLayout
 手动实现  Android 5.0 View 水纹动画  兼容5.0以下版本

### RippleLayout.class 
```
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Paint;
import android.support.annotation.Nullable;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;
import android.widget.LinearLayout;

import com.chuji.cjlive.R;

import java.util.ArrayList;

import static android.R.attr.x;

/**
 * Created by apple on 2017/3/23.
 */

public class RippleLayout extends LinearLayout {
    private Paint mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);

    private View targetView;
    private int INVALIDATE_DURATION = 20;
    private int[] mLocation = new int[2];
    // 当前需要绘制的 view
    private View mTargetView;
    private float mCenterX;
    private float mCenterY;
    private int mTargetWidth;
    private int mTargetHeight;
    private int minBetweenWidthAndHeight;
    private int mRevealRadiusGap;
    private int mRevealRadius;
    private boolean mIsPressed;
    private boolean mShouldDoAnimation;
    private int mMaxRadius;
    private boolean onOneView;

    private DispatchUpTouchEventRunnable mDispatchUpTouchEventRunnable = new DispatchUpTouchEventRunnable();

    public RippleLayout(Context context) {
        this(context, null);
    }

    public RippleLayout(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }


    public RippleLayout(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs);
    }

    private void init() {
        // 设置画笔颜色
        mPaint.setColor(getResources().getColor(R.color.clickback));
    }


    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);
        this.getLocationOnScreen(mLocation);
        System.out.println("mLocation[0]==" + mLocation[0]);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //getRawX 是相对于屏幕的绝对坐标
        int x = (int) ev.getRawX();
        int y = (int) ev.getRawY();
        int action = ev.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                //根据手指的位置确定是落在那个view上
                targetView = getTargetView(this, x, y);
                if (targetView != null && targetView.isEnabled()) {
                    mTargetView = targetView;
                    //初始化参数所点击的view
                    initParametersForChild(ev, targetView);

                    //延迟4毫秒 刷新  delayMilliseconds
                    postInvalidateDelayed(INVALIDATE_DURATION);
                }
            case MotionEvent.ACTION_UP:

                mIsPressed = false;
                postInvalidateDelayed(INVALIDATE_DURATION);

                mDispatchUpTouchEventRunnable.event = ev;
                //延迟处理up
                postDelayed(mDispatchUpTouchEventRunnable, 400);
                return true;
            case MotionEvent.ACTION_CANCEL:
                mIsPressed = false;
                postInvalidateDelayed(INVALIDATE_DURATION);

                break;
        }

        return super.dispatchTouchEvent(ev);
    }


    private class DispatchUpTouchEventRunnable implements Runnable {
        public MotionEvent event;

        @Override
        public void run() {
            // 为空 或者 不能点击
            if (mTargetView == null || !mTargetView.isEnabled()) {
                return;
            }
            //点击是 这个view以内
            if (isTouchPointView(mTargetView, (int) event.getRawX(), (int) event.getRawY())) {
                mTargetView.dispatchTouchEvent(event);
            }
        }
    }


    /**
     * View的绘制流程  先绘制背景， 在绘制自己(onDraw) 接着绘制子元素(dispatchDraw) 最后绘制一些装饰滚动条(onDrawScrollBars)
     * 为了适配Android L 子元素绘制
     */

    @Override
    protected void dispatchDraw(Canvas canvas) {
        super.dispatchDraw(canvas);
        //如果动画停止  手机所在的view自身的宽度 或者这个TargetView 不存在
        if (!mShouldDoAnimation || mTargetView == null || mTargetWidth <= 0) {
            return;
        }

        //如果圆的当前半径 超过了按钮 最小的宽度或者高度 的 一半， 则半径增加幅度变大；
        // 简单说就是 当一个按钮高度小于宽度时候 水纹扩散到顶部的时候，半径幅度变大。
        if (mRevealRadius > minBetweenWidthAndHeight / 2) {
            mRevealRadius += mRevealRadiusGap * 4;
        } else {
            mRevealRadius += mRevealRadiusGap;
        }

        int[] location = new int[2];
        mTargetView.getLocationOnScreen(location);

        //局部绘制 计算当前目标view 的 l t r b
        int left = location[0] - mLocation[0];
        int top = location[1] - mLocation[1];

        int right = left + mTargetView.getMeasuredWidth();
        int bottom = top + mTargetView.getMeasuredHeight();


        // canvas.save  保存状态，  再次canvas.restore 取出滑动。不受前面绘制的影响
        canvas.save();
        //绘制区域
        canvas.clipRect(left, top, right, bottom);
        canvas.drawCircle(mCenterX, mCenterY, mRevealRadius, mPaint);
        canvas.restore();

        if (mRevealRadius <= mMaxRadius) {
            postInvalidateDelayed(INVALIDATE_DURATION, left, top, right, bottom);
        } else if (!mIsPressed) {
            // 当绘制完成时候执行， 让button恢复
            mShouldDoAnimation = false;
            postInvalidateDelayed(INVALIDATE_DURATION, left, top, right, bottom);
            // 对外 实现 点击事件的效果，等button 刷新完成后执行
            if (onCompleteListener != null)
                onCompleteListener.onComplete(mTargetView.getId());
        }
    }

    private OnRippleCompleteListener onCompleteListener;

    public interface OnRippleCompleteListener {
        void onComplete(int id);
    }

    public void setOnRippleCompleteListener(OnRippleCompleteListener listener) {
        this.onCompleteListener = listener;
    }

    private void initParametersForChild(MotionEvent ev, View view) {
        // getX 相对于本view的距离  获取到点击点
        //ev 是ripple上的 所以以ripple 原点为准
        mCenterX = ev.getX();
        mCenterY = ev.getY();
        float viewX = view.getX();

        //手指所在的view自身的宽度
        mTargetWidth = view.getMeasuredWidth();
        mTargetHeight = view.getMeasuredHeight();

        //判断高度宽度最小值
        minBetweenWidthAndHeight = Math.min(mTargetWidth, mTargetHeight);
        mRevealRadius = 0;
        mRevealRadiusGap = minBetweenWidthAndHeight / 8;
        mIsPressed = true;
        mShouldDoAnimation = true;
        int[] location = new int[2];
        view.getLocationOnScreen(location);

        //left 是ripple 到屏幕的左边距
        //top  是ripple 到屏幕的顶边距
        int left = location[0] - mLocation[0];
        int top = location[1] - mLocation[1];
        System.out.println("location[0]=" + location[0] + "  mLocation[0]=" + mLocation[0] + " mCenterX=" + mCenterX + " viewX=" + viewX);

        //view距离左边界的距离
        int mTransformedCenterX = (int) mCenterX - left;
        //view距离上边界的距离
        int mTransformedCenterY = (int) mCenterX - top;

        int maxX = Math.max(mTransformedCenterX, mTargetWidth - mTransformedCenterX);
        int maxY = Math.max(mTransformedCenterY, mTargetHeight - mTransformedCenterY);
        mMaxRadius = Math.max(maxX, maxY);
    }

    private View getTargetView(View view, int x, int y) {
        View target = null;
        //判断view是否可以聚焦
        ArrayList<View> views = view.getTouchables();
        for (View child : views) {
            if (isTouchPointView(child, x, y)) {
                target = child;
                break;
            }
        }
        return target;
    }

    private boolean isTouchPointView(View child, int x, int y) {
        int[] locations = new int[2];
        child.getLocationOnScreen(locations);
        int top = locations[1];
        int left = locations[0];
        int right = left + child.getMeasuredWidth();
        int bottom = top + child.getMeasuredHeight();
        if (left <= x && x <= right && top <= y && y <= bottom && child.isClickable()) {
            return true;
        } else {
            return false;
        }
    }
}

```


### 布局中 包括需要点击效果的view 
 ** 对了RelativeLayout 默认clickable为false的空间 需要手动设置Clickable="true" **
```
 <com.chuji.cjlive.ui.RippleLayout
        android:id="@+id/rpl_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <RelativeLayout
            android:id="@+id/rl_romm_img"
            android:layout_width="match_parent"
            android:layout_height="65dp"
            android:layout_marginTop="15dp"
            android:background="@android:color/white"
            android:paddingLeft="15dp"
            android:clickable="true"
            android:paddingRight="10dp">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_centerVertical="true"
                android:text="微播间图标"
                android:textColor="@color/et_color"
                android:textSize="16sp" />
        </RelativeLayout>
</com.chuji.cjlive.ui.RippleLayout>  
```

### Activity 
```
   private RippleLayout rpl_content;
   rpl_content = (RippleLayout) findViewById(R.id.rpl_content);
   //点击事件写法
   rpl_content.setOnRippleCompleteListener(new RippleLayout.OnRippleCompleteListener() {
            @Override
            public void onComplete(int id) {
                Intent editIntent;
                switch (id) {
                  case R.id.rl_romm_img:
                        Intent intent = new Intent(SetActvitiy.this, ChangeHeadAct.class);
                        intent.putExtra("IMAGE", image);
                        startActivityForResult(intent, ROMM_IMG_CODE);
                        break;
                }
```
