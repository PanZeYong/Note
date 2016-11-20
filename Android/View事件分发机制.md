&emsp;&emsp;当我们点击控件时，就会做出相应的响应。那么这里面的原理是啥呢？其实这是事件分发机制，这既是核心知识点又是难点。当点击事件时，作用的对象是MotionEvent,然后该对象就会对事件进行分发，事件分发由三个方法共同作用：dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent。以下先从View的事件分发机制开始讲起。

### View事件分发机制
&emsp;&emsp;先来看一个简单demo，代码如下  

CustomButton.java

``` java
public class CustomButton extends Button {
    private static final String TAG = Button.class.getSimpleName();

    public CustomButton(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.d(TAG, "dispatchTouchEvent Down");
                break;

            case MotionEvent.ACTION_MOVE:
                Log.d(TAG, "dispatchTouchEvent Move");
                break;

            case MotionEvent.ACTION_UP:
                Log.d(TAG, "dispatchTouchEvent UP");
                break;

            default:
                break;
        }

        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.d(TAG, "onTouchEvent Down");
                break;

            case MotionEvent.ACTION_MOVE:
                Log.d(TAG, "onTouchEvent Move");
                break;

            case MotionEvent.ACTION_UP:
                Log.d(TAG, "onTouchEvent Up");
                break;
            default:
                break;
        }

        return super.onTouchEvent(event);
    }
}
```
MainActivity.java

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = MainActivity.class.getSimpleName();

    private Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        init();
        registerListener();
    }

    private void init() {
        mButton = (CustomButton) findViewById(R.id.button);
    }

    private void registerListener() {
        mButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "onClick");
            }
        });

        mButton.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.d(TAG, "onTouch");
                return true;
            }
        });
    }
}
```
activity.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.pan.vieweventdispatchdemo.MainActivity">

    <com.pan.vieweventdispatchdemo.CustomButton
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:text="View事件分发机制"/>
</RelativeLayout>
```
运行该程序，然后点击按钮，打印日志如下：

![](/Users/Pan/Github/Note/Android/Images/logcat_one.png)

可见，我们注册的监听器的回调方法都被调用了。并且**onTouch先于onClick**被调用，说明**onTouch优先级高于onClck**<font color="red">（**此处标记为问题1**）</font>。同时，**onTouch先于onTouchEvent**被调用，也说明**onTouch优先级高于onTouchEvent**<font color="red">（**此处标记为问题2**）</font>。那么，onClick的调用与onTouchEvent的调用有没有关系呢？我们可以来测试下，取消onClikcListener监听器注册，看看日志：

![](/Users/Pan/Github/Note/Android/Images/logcat_three.png)

从日志可以得知，**onClick**方法没有被调用，这是肯定的；但**onTouchEvent**方法依然被调用。那么有没有一种可能是onClick在onTouchEvent**满足一定条件被调用**呢<font color="red">（**此处标记为问题3**）</font>，想要知道答案，唯独看源码才知道，下面会有源码解析的。

&emsp;&emsp;不过不知你们有没有注意到，**onTouch**是有返回值的，上面是返回false；如果改为返回true，结果又会是怎样呢？再次运行程序，然后点击按钮，打印日志如下：

![](/Users/Pan/Github/Note/Android/Images/logcat_two.png)

由日志可知，我们注册onClickListener监听器的回调方法onClick和onTouchEvent方法都没有被调用，说明onTouch的返回值会关系到onTouchEvent和onClick这两个方法是否被调用<font color="red">（**此处标记为问题4**）。</font>那么要解决以上几个问题，唯独从源码入手才能找到答案。那么该从哪里入手呢？从日志可以看出，都是先调用dispatchTouchEvent方法，很显然从该方法入手。所以跳到Button类查找，结果发现没有；再跳到Button的父类TextView查找也没有，也没有；最后跳到View，终于找到。源码如下：

##### View dispatchTouchEvent方法解析
```java
public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
	if (event.isTargetAccessibilityFocus()) {
		// We don't have focus or no virtual descendant has it, do not handle the event.
		if (!isAccessibilityFocusedViewOrHost()) {
			return false;
     	}
       // We have focus and got the event, then use normal event dispatch.
		event.setTargetAccessibilityFocus(false);
	}

	boolean result = false;

	if (mInputEventConsistencyVerifier != null) {
		mInputEventConsistencyVerifier.onTouchEvent(event, 0);
	}

	final int actionMasked = event.getActionMasked();
	if (actionMasked == MotionEvent.ACTION_DOWN) {
		// Defensive cleanup for new gesture
		stopNestedScroll();
	}

	if (onFilterTouchEventForSecurity(event)) {
		if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
			result = true;
		}
		//noinspection SimplifiableIfStatement
		ListenerInfo li = mListenerInfo;
		if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED && li.mOnTouchListener.onTouch(this, event)) {
				result = true;
 		}

		if (!result && onTouchEvent(event)) {
			result = true;
		}
	}

	if (!result && mInputEventConsistencyVerifier != null) {
		mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
	}

	// Clean up after nested scrolls if this is the end of a gesture;
	// also cancel it if we tried an ACTION_DOWN but we didn't want the rest
	// of the gesture.
	if (actionMasked == MotionEvent.ACTION_UP ||actionMasked == MotionEvent.ACTION_CANCEL ||(actionMasked == MotionEvent.ACTION_DOWN && !result)) {
		stopNestedScroll();
	}

	return result;
}
```
第14-16行代码主要是安全验证；重点来看第24-37行代码，第24行代码主要是对事件进行安全过滤；第30行是if语句，有**四**个条件,如果这四个条件都为ture的话，result变量被赋值为**true**,同时也作为返回值返回。那么这四个条件具体是啥呢？

- `li != null`：第29行已经对li变量进行赋值了，那么**mListenerInfo**这个变量又是怎么被赋值呢？经过一番查找，在View找到**getListenerInfo()**方法，代码如下
	
	```java
	ListenerInfo getListenerInfo() {        if (mListenerInfo != null) {            return mListenerInfo;        }        mListenerInfo = new ListenerInfo();        return mListenerInfo;    }
	```
	
	也就是说，一但这个方法getListenerInfo()被调用的话，变量**mListenerInfo**就被赋值，即不为null，条件成立，为ture；否则为false。
- `li.mOnTouchListener != null`：那么**mOnTouchListener**是怎么赋值的呢？经过一番查找，在View里找到方法setOnTouchListener()方法，代码如下

	```java
	/**     * Register a callback to be invoked when a touch event is sent to this view.     * @param l the touch listener to attach to this view     */
	
	public void setOnTouchListener(OnTouchListener l) {        getListenerInfo().mOnTouchListener = l;    }
	```
	
	也就是说，一旦我们注册监听器**OnTouchListener**的话，变量**mOnTouchListener**就会被赋值，即不为null,条件成立为true,同时也该方法也调用方法**getListenerInfo()**,这也就使变量**mListenerInfo**被赋值，即变量**li**不为null，条件1成立；否则，为false。
- `(mViewFlags & ENABLED_MASK) == ENABLED`：这个条件主要判断控件的状态，即控件是否处于**enabled**状态。例如，Button默认值为true，而TextView、ImageView默认值为false。
- `li.mOnTouchListener.onTouch(this, event)`：该条件是根据onTouch()方法的返回值来决定，而该方法是接口OnTouchListener的回调方法，默认是空方法，需要我们自己重写。如果我们返回true的话，该条件成立，即true；如果我们返回false的话，该条件不成立，即false。

根据这四个条件的分析，我们可以用它们来解决以上遗留的问题。如果这四个条件其中有一个为false时，第34-36行才会被执行，**onTouchEvent**方法才会被调用。如果这四个条件都为true的话，**onTouchEvent**方法就不会被调用。这也就解决了**问题2**。那么调用了onTouchEvent，它又是何方神圣呢？找到源码如下：

##### View onTouchEvent方法解析

```java
public boolean onTouchEvent(MotionEvent event) {	final float x = event.getX();	final float y = event.getY();	final int viewFlags = mViewFlags;	final int action = event.getAction();	if ((viewFlags & ENABLED_MASK) == DISABLED) {		if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {			setPressed(false);		}	// A disabled view that is clickable still consumes the touch     // events, it just doesn't respond to them.     	return (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);	}	if (mTouchDelegate != null) {		if (mTouchDelegate.onTouchEvent(event)) {			return true;		}	}
		if (((viewFlags & CLICKABLE) == CLICKABLE ||(viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||(viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {		switch (action) {			case MotionEvent.ACTION_UP:				boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;				if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {					// take focus if we don't have it already and we should in					// touch mode.             	boolean focusTaken = false;             	if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {					focusTaken = requestFocus();				}				if (prepressed) {					// The button is being released before we actually					// showed it as pressed.  Make it show the pressed					// state now (before scheduling the click) to ensure					// the user sees it.					setPressed(true, x, y);				}                 if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {					// This is a tap, so remove the longpress check             		removeLongPressCallback();                   	// Only perform take click actions if we were in the pressed state                   	if (!focusTaken) {                     	// Use a Runnable and post this rather than calling                        // performClick directly. This lets other visual state                        // of the view update before click actions start.                        if (mPerformClick == null) {                        	mPerformClick = new PerformClick();                         }
                                                  if (!post(mPerformClick)) {                         	performClick();                         }                     }              	  }
              	                    if (mUnsetPressedState == null) {						mUnsetPressedState = new UnsetPressedState(); 					}
 									if (prepressed) {					postDelayed(mUnsetPressedState,                                   ViewConfiguration.getPressedStateDuration());                 } else if (!post(mUnsetPressedState)) {                            // If the post failed, unpress right now							mUnsetPressedState.run();				 }					removeTapCallback();				}                    mIgnoreNextUpEvent = false;                    break;
                    				case MotionEvent.ACTION_DOWN:                    mHasPerformedLongPress = false;                    if (performButtonActionOnTouchDown(event)) {                        break;                    }                    // Walk up the hierarchy to determine if we're inside a scrolling container.                    boolean isInScrollingContainer = isInScrollingContainer();                    // For views inside a scrolling container, delay the pressed feedback for                    // a short period in case this is a scroll.                    if (isInScrollingContainer) {                        mPrivateFlags |= PFLAG_PREPRESSED;                        if (mPendingCheckForTap == null) {                            mPendingCheckForTap = new CheckForTap();                        }                        mPendingCheckForTap.x = event.getX();                        mPendingCheckForTap.y = event.getY();                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());                    } else {                        // Not inside a scrolling container, so show the feedback right away                        setPressed(true, x, y);                        checkForLongClick(0, x, y);                    }                    break;
                                    case MotionEvent.ACTION_CANCEL:                    setPressed(false);                    removeTapCallback();                    removeLongPressCallback();                    mInContextButtonPress = false;                    mHasPerformedLongPress = false;                    mIgnoreNextUpEvent = false;                    break;
                                    case MotionEvent.ACTION_MOVE:                    drawableHotspotChanged(x, y);                    // Be lenient about moving outside of buttons                    if (!pointInView(x, y, mTouchSlop)) {                        // Outside button                        removeTapCallback();                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {                            // Remove any future long press/tap checks                            removeLongPressCallback();                            setPressed(false);                        }                    }                    break;            }            return true;        }        return false;    }
```
第6-13行代码主要功能是即使控件处于**disabled状态**，也能对**消费事件**，只是不会做出反应，不会影响**ouTouchEvent**返回值。从第20行代码开始是对点击事件的具体处理，从if语句可以看出，只要View的**CLICKABLE**和**LONG_CLICKABLE**其中一个为**true**，**onTounchEvent**就会返回**true**，即**消费事件**。而**CLICKABLE默认为true，LONG_CLICKABLE默认为false**，因此**ouTouchEvent默认返回true**，即消费事件；除非View被设置为不可点击状态（clickable）和LONG_CLICKABLE同时为false，onTouchEvent才返回false.	进入if语句后，switch语句对点击事件action进行判断，分别有ACTION_DOWN、ACTION_MOVE、ACTION_UP、ACTION_CANCEL.看第38-54行，当处于ACTION_UP状态时，会调用performClick()方法，那么该方法具体执行什么操作呢，定位到该方法源码

```java
/**
     * Call this view's OnClickListener, if it is defined.  Performs all normal
     * actions associated with clicking: reporting accessibility event, playing
     * a sound, etc.
     *
     * @return True there was an assigned OnClickListener that was called, false
     *         otherwise is returned.
     */
public boolean performClick() {
	final boolean result;
	final ListenerInfo li = mListenerInfo;
	if (li != null && li.mOnClickListener != null) {
		playSoundEffect(SoundEffectConstants.CLICK);
		i.mOnClickListener.onClick(this);
		result = true;
	} else {
		result = false;
	}
	sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
	return result;
}
```
定位到第2行代码，该if语句有两个条件：

- `li != null`：分析如上

- `li.mOnclickListener != null`：那么mOnclickListener是怎么被赋值呢？经过一番查找，找到方法**setOnClickListener()**，源码如下

```java
/**
     * Register a callback to be invoked when this view is clicked. If this view is not
     * clickable, it becomes clickable.
     *
     * @param l The callback that will run
     *
     * @see #setClickable(boolean)
     */
public void setOnClickListener(@Nullable OnClickListener l) {
	if (!isClickable()) {
		setClickable(true);
	}
	getListenerInfo().mOnClickListener = l;
}
```

先判断View是否处于可点击状态，然后再对变量**mOnClickListener**赋值。也就是说，一旦我们注册OnClickListener监听器，**mOnClickListener**就不为null，以上if语句条件就成立，**OnClickListener**接口的回调方法**onClick()**就会被调用，而该方法又是空实现，需要我们自己重写，实现自己的逻辑。到这里，基本都解决以上遗留的问题了。

小结

- 首先调用方法dispatchTouchEvent，如果设置onTouchListener监听器并且OnTouchListener.onTouch方法返回true，onTouchEvent方法不会被调用；如果OnTouchListener.onTouch返回false,onTouchEvent()方法返回false,onTouchEvent方法会被调用，onTouch优先级高于onTouchEvent

- 如果onTouchEvent方法被调用，并且设置OnClickListener监听器，OnClickListener.onClick方法会被调用。可见onClick方法优先级是最低的。

- 不管View是否处于可见状态（enable），只要CLICKABLE和LONG_CLICKABLE其中一个返回true，onTouchEvent方法就返回true，即事件被消费。而CLICKABLE默认为true，LONG_CLICKABLE默认为false，因此ouTouchEvent默认返回true。

&emsp;&emsp;以上只是对单独View进行事件分发机制进行分析，那么如果有多个View呢？下面就来分析下ViewGroup事件分发机制。

### ViewGroup事件分发机制

老规矩，还是从一个简单的demo说起，代码如下
CustomLayout.java

```java
public class CustomLayout extends LinearLayout {
    private static final String TAG = CustomLayout.class.getSimpleName();

    public CustomLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.d(TAG, "dispatchTouchEvent Down");
                break;

            case MotionEvent.ACTION_MOVE:
                Log.d(TAG, "dispatchTouchEvent Move");
                break;

            case MotionEvent.ACTION_UP:
                Log.d(TAG, "dispatchTouchEvent UP");
                break;

            default:
                break;
        }

        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        Log.d(TAG, "onInterceptTouchEvent");
        return super.onInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.d(TAG, "onTouchEvent Down");
                break;

            case MotionEvent.ACTION_MOVE:
                Log.d(TAG, "onTouchEvent Move");
                break;

            case MotionEvent.ACTION_UP:
                Log.d(TAG, "onTouchEvent Up");
                break;

            default:
                break;
        }

        return super.onTouchEvent(event);
    }
}
```
MainActivity.java

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = MainActivity.class.getSimpleName();

    private CustomButton mButtonOne;
    private CustomButton mButtonTwo;
    private CustomLayout mLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_view_group);

        init();
        registerListener();
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.d(TAG, "dispatchTouchEvent Down");
                break;

            case MotionEvent.ACTION_MOVE:
                Log.d(TAG, "dispatchTouchEvent Move");
                break;

            case MotionEvent.ACTION_UP:
                Log.d(TAG, "dispatchTouchEvent UP");
                break;

            default:
                break;
        }

        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.d(TAG, "onTouchEvent Down");
                break;

            case MotionEvent.ACTION_MOVE:
                Log.d(TAG, "onTouchEvent Move");
                break;

            case MotionEvent.ACTION_UP:
                Log.d(TAG, "onTouchEvent UP");
                break;

            default:
                break;
        }

        return super.onTouchEvent(event);
    }

    private void init() {
        mButtonOne = (CustomButton) findViewById(R.id.btn_one);
        mButtonTwo = (CustomButton) findViewById(R.id.btn_two);
        mLayout = (CustomLayout) findViewById(R.id.parent);
    }

    private void registerListener() {
        mButtonOne.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "Button One onClick");
            }
        });

        mButtonOne.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.d(TAG, "Button One onTouch");
                return false;
            }
        });

        mButtonTwo.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "Button Two onClick");
            }
        });

        mButtonTwo.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.d(TAG, "Button Two onTouch");
                return false;
            }
        });

        mLayout.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "CustomLayout onClick");
            }
        });

        mLayout.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.d(TAG, "CustomLayout onTouch");
                return false;
            }
        });
    }
}
```

activity_view_group.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<com.pan.vieweventdispatchdemo.CustomLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/parent"
    android:orientation="vertical"
    android:gravity="center"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="250dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:layout_marginBottom="20dp">
        <com.pan.vieweventdispatchdemo.CustomButton
            android:id="@+id/btn_one"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Button 1" />
    </LinearLayout>

    <com.pan.vieweventdispatchdemo.CustomButton
        android:id="@+id/btn_two"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Button 2"/>
</com.pan.vieweventdispatchdemo.CustomLayout>
```

运行效果如下

<center>![](/Users/Pan/Github/Note/Android/Images/result.png)</center>

接下来看不同种情况下的点击事件

- 点击BUTTON 1,打印日志如下：

	![](/Users/Pan/Github/Note/Android/Images/logcat_four.png)
	
- 点击BUTTON 2,打印日志如下：
	
	![](/Users/Pan/Github/Note/Android/Images/logcat_five.png)
	
- 点击蓝色区域，打印日志如下

	![](/Users/Pan/Github/Note/Android/Images/logcat_six.png)

- 点击白色区域，打印日志如下

	![](/Users/Pan/Github/Note/Android/Images/logcat_seven.png)
	
&emsp;&emsp;从以上可以看出，每当点击View时，都会先调用Activity方法dispatchTouchEvent，接着调用View父布局方法dispatchTouchEvent，再调用View方法dispatchTouchEvent，接下来方法调用就跟之前分析View一样。可以看出，View的事件分发是从Activity开始，然后传递给View父布局，再传递给View本身？那么如果在某一阶段对事件不分发，又会出现怎样的结果呢？先从View开始测试，假如BUTTON 2方法dispatchTouchEvent返回false，打印日志如下

![](/Users/Pan/Github/Note/Android/Images/logcat_eight.png)

当事件传递到View时，如果View不对事件进行分发，即不消费事件时，事件又会再回传给它的父布局，由父布局对事件消费。那如果父布局不对事件进行消费，事件应该是回传给Activity进行消费。是不是这样呢，以下通过源码分析会给出答案的<font color="red">**问题5**</font>。

&emsp;&emsp;不知有没有发现，这里多了方法onInterceptTouchEvent方法调用，该方法只有ViewGroup才有，View没有；并且该方法有返回值，数据类型为boolean，那么该方法不同的返回值又会产生怎样的影响呢？以下会揭晓的。

&emsp;&emsp;为了弄清楚事件怎么被传递到ViewGroup，再传到View，先从Activity方法dispatchTouchEvent方法看起，源码如下：
	
```java
/**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (ev.getAction() == MotionEvent.ACTION_DOWN) {
		onUserInteraction();
	}
	
	if (getWindow().superDispatchTouchEvent(ev)) {
		return true;
	}
	
	return onTouchEvent(ev);
}
```

第2-4行代码大概是事件出ACTION_DOWN状态时，调用**onUserInteraction**，该方法是空实现，需要的话可以重写，调用时机是Activity退到后台时会被调用。接着再看第16-18行代码，首先事件交给Activity所附属的Window进行分发，如果`getWindow().superDispatchTouchEvent(ev)`返回true，Activity方法dispatchTouchEvent返回true，对事件进行分发，即消费事件，整个事件循环也就结束；如果返回false，所有View的onTouchEvent会被调用，Activity的onTouchEvent方法也会被调用。那么Window方法superDispatchTouchEvent具体实现是什么呢？得来看它的代码。由于Window是抽象类，得找到它的实现类，而它的实现类时PhoneWindow，定位到superDispatchTouchEvent方法，源码如下：

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
	return mDecor.superDispatchTouchEvent(event);
}
```

很明显PhoneWindow直接将事件分发给mDecor，那么mDecor又是什么呢？之前写过的一篇笔记：Android学习笔记：浅析setContentView，该笔记有记录过。这里稍微说下，mDecor是Decor对象，通过`getWindow().getDecorView()`可以获取到mDecor对象，通过`findViewById(R.id.content)`可以获取到我们通过setContentView设置View,也就是说，我们平时通过setContentView所设置的View是DecorView的子类。那么接下来看` mDecor.superDispatchTouchEvent(event);`具体实现

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
	return super.dispatchTouchEvent(event);
}
```

该方法调用父类dispatchTouchEvent方法，而DecorView的父类是FrameLayout,所以到FrameLayout类查找有没有dispatchTouchEvent方法，结果发现没有；那么到FrameLayout的父类ViewGroup查找，结果找到了，一切的奥秘都在这个方法里。

##### ViewGroup dispatchTouchEvent方法解析

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (mInputEventConsistencyVerifier != null){
		mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
	}
	// If the event targets the accessibility focused view and this is it, start
	// normal event dispatch. Maybe a descendant is what will handle the click.
	if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
		ev.setTargetAccessibilityFocus(false);
	}
	
	boolean handled = false;
	
	if (onFilterTouchEventForSecurity(ev)) {
		final int action = ev.getAction();
		final int actionMasked = action & MotionEvent.ACTION_MASK;
		// Handle an initial down.
		if (actionMasked == MotionEvent.ACTION_DOWN) {
			// Throw away all previous state when starting a new touch gesture.
			// The framework may have dropped the up or cancel event for the previous gesture
			// due to an app switch, ANR, or some other state change.
			cancelAndClearTouchTargets(ev);
			resetTouchState();
		}
		
		// Check for interception.
		final boolean intercepted;
		if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
			final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
			if (!disallowIntercept) {
				intercepted = onInterceptTouchEvent(ev);
				ev.setAction(action); // restore action in case it was changed
			} else {
				intercepted = false;
			}
		} else {
			// There are no touch targets and this action is not an initial down
			// so this view group continues to intercept touches.
			intercepted = true;
		}
		
		// If intercepted, start normal event dispatch. Also if there is already
		// a view that is handling the gesture, do normal event dispatch.
		if (intercepted || mFirstTouchTarget != null) {
			ev.setTargetAccessibilityFocus(false);
		}
		// Check for cancelation.
		final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
		// Update list of touch targets for pointer down, if needed.
		final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
		TouchTarget newTouchTarget = null;
		boolean alreadyDispatchedToNewTouchTarget = false;
		if (!canceled && !intercepted) {
			// If the event is targeting accessiiblity focus we give it to the
			// view that has accessibility focus and if it does not handle it
			// we clear the flag and dispatch the event to all children as usual.
			// We are looking up the accessibility focused host to avoid keeping
			// state since these events are very rare.
			View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus() ? findChildWithAccessibilityFocus() : null;
			
				if (actionMasked == MotionEvent.ACTION_DOWN || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
					final int actionIndex = ev.getActionIndex(); // always 0 for down
					final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex) : TouchTarget.ALL_POINTER_IDS;
					// Clean up earlier touch targets for this pointer id in case they
					// have become out of sync.
					removePointersFromTouchTargets(idBitsToAssign);
					final int childrenCount = mChildrenCount;
					
					if (newTouchTarget == null && childrenCount != 0) {
						final float x = ev.getX(actionIndex);
						final float y = ev.getY(actionIndex);
						// Find a child that can receive the event.
						// Scan children from front to back.
						final ArrayList<View> preorderedList = buildTouchDispatchChildList();
						final boolean customOrder = preorderedList == null && isChildrenDrawingOrderEnabled();
						final View[] children = mChildren;
						
						for (int i = childrenCount - 1; i >= 0; i--) {
							final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
							final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
							// If there is a view that has accessibility focus we want it
							// to get the event first and if not handled we will perform a
							// normal dispatch. We may do a double iteration but this is
							// safer given the timeframe.
							if (childWithAccessibilityFocus != null) {
								if (childWithAccessibilityFocus != child) {
									continue;
								}
								
								childWithAccessibilityFocus = null;
								i = childrenCount - 1;
							}
							
							if(canViewReceivePointerEvents(child)
|| !isTransformedTouchPointInView(x, y, child, null)) {
								ev.setTargetAccessibilityFocus(false);
								continue;
							}
							
							newTouchTarget = getTouchTarget(child);
							if (newTouchTarget != null) {
								// Child is already receiving touch within its bounds.
								// Give it the new pointer in addition to the ones it is handling.                               								newTouchTarget.pointerIdBits |= idBitsToAssign;
								break;
							}
                            									resetCancelNextUpFlag(child);
                            		
							if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
								// Child wants to receive touch within its bounds.
								mLastTouchDownTime = ev.getDownTime();
								if (preorderedList != null) {
									// childIndex points into presorted list, find original index
									for (int j = 0; j < childrenCount; j++) {
										if (children[childIndex] == mChildren[j]) {
											mLastTouchDownIndex = j;
											break;
										}
									}
								} else {
									mLastTouchDownIndex = childIndex;
								}
								
							mLastTouchDownX = ev.getX();
							mLastTouchDownY = ev.getY();
							newTouchTarget = addTouchTarget(child, idBitsToAssign);                                							alreadyDispatchedToNewTouchTarget = true;
							break;
						}
						
						// The accessibility focus didn't handle the event, so clear
						// the flag and do a normal dispatch to all children.                            						ev.setTargetAccessibilityFocus(false);
					}
					
					if (preorderedList != null) {
						preorderedList.clear();
					}
					
					if (newTouchTarget == null && mFirstTouchTarget != null) {
						// Did not find a child to receive the event.
						// Assign the pointer to the least recently added target.
						newTouchTarget = mFirstTouchTarget;
						while (newTouchTarget.next != null) {
							newTouchTarget = newTouchTarget.next;
 						}
						newTouchTarget.pointerIdBits |= idBitsToAssign;
					}
				}
			}
			
			// Dispatch to touch targets.
			if (mFirstTouchTarget == null) {
				// No touch targets so treat this as an ordinary view.
				handled = dispatchTransformedTouchEvent(ev, canceled, null,
				TouchTarget.ALL_POINTER_IDS);
			} else {
				// Dispatch to touch targets, excluding the new touch target if we already
				// dispatched to it.  Cancel touch targets if necessary.
				TouchTarget predecessor = null;
				TouchTarget target = mFirstTouchTarget;
				while (target != null) {
					final TouchTarget next = target.next;
					
					if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
						handled = true;
					} else {
						final boolean cancelChild = resetCancelNextUpFlag(target.child) || intercepted;
						if (dispatchTransformedTouchEvent(ev, cancelChild,target.child, target.pointerIdBits)) {
							handled = true;
						}
						
						if (cancelChild) {
							if (predecessor == null) {
								mFirstTouchTarget = next;
							} else {
								predecessor.next = next;
							}
							target.recycle();
							target = next;
							continue;
						}
					}
					
					predecessor = target;
					target = next;
				}
			}
			
			// Update list of touch targets for pointer up or cancel, if needed.
			if (canceled || actionMasked == MotionEvent.ACTION_UP || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
				resetTouchState();
			} else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
				final int actionIndex = ev.getActionIndex();
				final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);               				removePointersFromTouchTargets(idBitsToRemove);
			}
		}
		
		if (!handled && mInputEventConsistencyVerifier != null) {
			mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
		}
		return handled;
}
```
从第26行看起，从注释可以看出检查是否拦截。从if语句可以看出有两个条件：

- `actionMasked == MotionEvent.ACTION_DOWN`
- `mFirstTouchTarget != null`

&emsp;&emsp;第一个条件比较好理解，至于变量**mFirstTouchTarget**是什么呢？往下看就会知道的。如果为true的话，就会进入if语句。标志位`**FLAG_DISALLOW_INTERCEPT** `表示是否禁用拦截，可以通过**requestDisallowInterceptTouchEvent**方法来设置，一般用于View。与变量**mGroupFlags**进行`&`操作后赋值给变量**disallowIntercept**。如果变量**disallowIntercept**为true，不允许禁用拦截，变量**intercepted**为false；如果为false，会调用ViewGroup方法**onInterceptTouchEvent(ev)**，该方法默认返回false，也就是说，ViewGroup默认不对事件进行拦截，变量**intercepted**同样被赋值为false。

&emsp;&emsp;第45行检查是否取消事件；第53行如果不取消和不拦截的话就会进入该if语句内部逻辑，第69-127行主要是遍历所有子View元素，判断子View是否接收到点击事件。那么衡量事件是否接收到点击事件的标准是什么呢？第94行可以看出，即**子元素是否在播放动画**和**点击事件的坐标是否落在子元素的区域**。如果都满足的话，就会执行第108行，即调用**dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)方法**，那么该方法又是啥呢，看下源码

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,View child, int desiredPointerIdBits) {
	final boolean handled;
	// Canceling motions is a special case.  We don't need to perform any transformations
	// or filtering.  The important part is the action, not the contents.
	final int oldAction = event.getAction();
	
	if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
		event.setAction(MotionEvent.ACTION_CANCEL);
		
		if (child == null) {
			handled = super.dispatchTouchEvent(event);
		} else {
			handled = child.dispatchTouchEvent(event);
		}
		
		event.setAction(oldAction);
		return handled;
	}
	
	// Calculate the number of pointers to deliver.
	final int oldPointerIdBits = event.getPointerIdBits();
	final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;
	// If for some reason we ended up in an inconsistent state where it looks like we
	// might produce a motion event with no pointers in it, then drop the event.
	if (newPointerIdBits == 0) {
		return false;
	}
	// If the number of pointers is the same and we don't need to perform any fancy
	// irreversible transformations, then we can reuse the motion event for this
	// dispatch as long as we are careful to revert any changes we make.
	// Otherwise we need to make a copy.
	final MotionEvent transformedEvent;
	if (newPointerIdBits == oldPointerIdBits) {
		if (child == null || child.hasIdentityMatrix()) {
			if (child == null) {
				handled = super.dispatchTouchEvent(event);
			} else {
				final float offsetX = mScrollX - child.mLeft;
				final float offsetY = mScrollY - child.mTop;
				event.offsetLocation(offsetX, offsetY);
				handled = child.dispatchTouchEvent(event);
				event.offsetLocation(-offsetX, -offsetY);
			}
			return handled;
		}
		transformedEvent = MotionEvent.obtain(event);
	} else {
		transformedEvent = event.split(newPointerIdBits);
	}
	
	// Perform any necessary transformations and dispatch.
	if (child == null) {
		handled = super.dispatchTouchEvent(transformedEvent);
	} else {
		final float offsetX = mScrollX - child.mLeft;
		final float offsetY = mScrollY - child.mTop;
		transformedEvent.offsetLocation(offsetX, offsetY);
		if (! child.hasIdentityMatrix()) {
		               transformedEvent.transform(child.getInverseMatrix());
		}
		handled = child.dispatchTouchEvent(transformedEvent);
	}
	
	// Done.
  transformedEvent.recycle();
  return handled;
}
```

有这么几行代码

```java
if (child == null) {
	handled = super.dispatchTouchEvent(event);
} else {
	handled = child.dispatchTouchEvent(event);
}
```

由上面可以知道，变量**mChild**不为空，所以会调用子View**dispatchTouchEvent方法**，至于View方法**dispatchTouchEvent**解析看上面。如果View方法**dispatchTouchEvent**返回true的话，就会进入该if语句，主要看第125行，调用方法`addTouchTarget(child, idBitsToAssign);`那么该方法是什么呢？进去瞧瞧

```java
 /**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
	final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
	target.next = mFirstTouchTarget;
	mFirstTouchTarget = target;
	return target;
}
```

原来是对变量**mFirestTouchTarget（自己对该变量理解不是很懂）**赋值。接下来看第150行代码，如果变量**mFirstTouchTarget**为null,说明没有合适的子View处理事件，有两种情况：**ViewGroup没有字元素**或者子View方法**dispatchTouchEvent**方法返回false。那么就会执行第152行，从以上方法** dispatchTransformedTouchEvent**源码可以看出会调用View方法**dispatchTouchEvent**；否则就会执行else语句。

&emsp;&emsp;小结

- 事件分发从Activity开始，先传到Activity所属Window，接着交给顶级DecorView，然后传递给我们通过setContentView方法锁设置的View（ViewGroup），最后由ViewGroup传递给子View。
- ViewGroup可以对事件进行拦截，即方法**onInterceptTouchEvent**返回true，拦截事件，消费事件，不会再向子View进行分发。
- 子View如果消费事件，就不会再传给ViewGroup;如果没有消费，就会传给ViewGroup;如果ViewGroup都没有对事件进行消费，最后由Activity消费。

&emsp;&emsp;最近在拜读任玉刚大神《Android 开发艺术探索》这本书第三章 View的事件体系和看了郭霖大神关于事件分发机制的博客（后面附上链接），对事件分发机制有了初步了解，为了方便以后自己查阅，所以就把它记录下来，作为笔记，该笔记有些观点来自书中和博客。由于自己的能力有限，对这知识点只是初步理解，如果有错误，希望指出，谢谢！

### 博客链接
#### [Android事件分发机制完全解析，带你从源码的角度彻底理解(上)](http://blog.csdn.net/sinyu890807/article/details/9097463)

#### [ Android事件分发机制完全解析，带你从源码的角度彻底理解(下)](http://blog.csdn.net/sinyu890807/article/details/9153747)
	



	
	




