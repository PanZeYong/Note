# 浅析setContentView
&emsp;&emsp;每次在Github上看到炫酷效果的demo时，都会想，什么时候自己也能实现这样的效果。要想有所收获，就必须得付出。于是，我决定花时间来学习自定义View，虽说自定义View水很深，但凡事得一步步来，终会有所收获的。要实现自定义View，必须得先了解View的绘制流程，但在学习View的绘制流程之前，先来了解下setContentView这个方法时如何将XML布局加载到当前Activity？

&emsp;&emsp;从[Activity](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.setContentView%28int%29)源码可以看出，setContentView方法有三个重载方法，分别是：

- [setContentView(int layoutResId)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.setContentView%28int%29)：将布局文件加载到当前Activity，并且在Activity充当父布局角色。

- [setContentView(View view)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.setContentView%28android.view.View%29)：将View加载到当前Activity。当调用这个方法设置View时，自己对View设置布局参数宽高会被忽略，默认的宽高是android.view.ViewGroup.LayoutParams.MATCH_PARENT。如果想使用自己定义的布局参数，可以使用setContentView(View view, ViewGroup.LayoutParams params)这个方法代替。

- [setContentView(View view, ViewGroup.LayoutParams params)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.setContentView%28android.view.View%2Candroid.view.ViewGroup.LayoutParams%29)：将View加载到当前Activity。  

从源码来看，这个三个重载方法的实现逻辑都差不多，所以我只分析[setContentView(int layoutResId)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.setContentView%28int%29).

####Step 1
&emsp;&emsp;进入到Activity源码，找到setContentView(int layoutResId)方法如下：

	public void setContentView(int layoutResID) {
       getWindow().setContentView(layoutResID);
       initWindowDecorActionBar();
   	}
只有两行代码，有木有一种挺简单的感觉。其实不然，往下看就知道有多复杂了。从第2行看起，进入[getWindow()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.getWindow%28%29)这个方法，代码如下
	
	public Window getWindow() {
		return mWindow;
    }
获取Activity当前[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)对象，[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)是抽象类，不能实例化对象，那么是怎么获取到[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)对象？在Activity源码中attach方法有这样的一行代码

	mWindow = PolicyManager.makeNewWindow(this);
很明显，这是获取Window对象的，继续往下看。进入[PolicyManager](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/PolicyManager.java#PolicyManager.makeNewWindow%28android.content.Context%29)类[makeNewWindow()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/PolicyManager.java#PolicyManager.makeNewWindow%28android.content.Context%29)方法

	public static Window makeNewWindow(Context context) {
        return sPolicy.makeNewWindow(context);
    }
那么sPolicy又是什么呢，代码如下：

	private static final String POLICY_IMPL_CLASS_NAME =
        "com.android.internal.policy.impl.Policy";
        
    private static final IPolicy sPolicy;
    
    static {
        // Pull in the actual implementation of the policy at run-time
        try {
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);
            sPolicy = (IPolicy)policyClass.newInstance();
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be loaded", ex);
        } catch (InstantiationException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        }
    }
从以上代码可以看出，[IPolicy](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/IPolicy.java#IPolicy)是接口，通过反射机制获取到sPolicy，该接口的实现类是[Policy](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/Policy.java#Policy)，所以定位到该类[makeNewWindow()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/Policy.java#Policy.makeNewWindow%28android.content.Context%29)方法

	public Window makeNewWindow(Context context) {
        return new PhoneWindow(context);
    }
终于获取到[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)对象，该对象的类型是[PhoneWindow](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow)。那么[PhoneWindow](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow)又是啥呢？ 进入[PhoneWindow](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow)该类一看，原来该类是继承抽象类[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)，并重写[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)类抽象方法。可以这么理解，[Activity](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/app/Activity.java#Activity.setContentView%28int%29)的窗口是[PhoneWindow](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow)。获取到[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)对象也就结束了，以下总结下获取[Window](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/view/Window.java#Window)对象流程：
&emsp;&emsp;**getWindow()  --->  PolicyManager.makeNewWindow(this)  --->  sPolicy.makeNewWindow(context)  --->  new PhoneWindow(context)**

####Step2
&emsp;&emsp;进入[PhoneWindow](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow)类[setContentView(int layoutResId)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.setContentView%28int%29)方法中

	@Override
    public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

		if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
			final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,getContext());
            transitionTo(newScene);
         } else {
             mLayoutInflater.inflate(layoutResID, mContentParent);
         }
         
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
第3行代码判断**mContentParent**(类型是**ViewGroup**)是否为null，由于一开始加载布局，所以**mContentParent**为null。于是执行if语句，即执行[installDecor()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.installDecor%28%29)方法，于是进入到该方法代码实现

	private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            mTitleView = (TextView)findViewById(com.android.internal.R.id.title);
            if (mTitleView != null) {
                if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                    View titleContainer = findViewById(com.android.internal.R.id.title_container);
                    if (titleContainer != null) {
                        titleContainer.setVisibility(View.GONE);
                    } else {
                        mTitleView.setVisibility(View.GONE);
                    }
                    if (mContentParent instanceof FrameLayout) {
                        ((FrameLayout)mContentParent).setForeground(null);
                    }
                } else {
                    mTitleView.setText(mTitle);
                }
            } else {
                mActionBar = (ActionBarView) findViewById(com.android.internal.R.id.action_bar);
                if (mActionBar != null) {
                    if (mActionBar.getTitle() == null) {
                        mActionBar.setWindowTitle(mTitle);
                    }
                    final int localFeatures = getLocalFeatures();
                    if ((localFeatures & (1 << FEATURE_PROGRESS)) != 0) {
                        mActionBar.initProgress();
                    }
                    if ((localFeatures & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
                        mActionBar.initIndeterminateProgress();
                    }
                    // Post the panel invalidate for later; avoid application onCreateOptionsMenu
                    // being called in the middle of onCreate or similar.
                    mDecor.post(new Runnable() {
                        public void run() {
                            if (!isDestroyed()) {
                                invalidatePanelMenu(FEATURE_ACTION_BAR);
                            }
                        }
                    });
                }
            }
        }
    }
这个方法有点长，慢慢来看吧。（PS：说真的，我不是百分之百理解，大概理解流程而已）。刚进入，就判断**mDecor**（类型为[DecorView](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.DecorView)）是否为null。由于一开始**mDecor**为null，所以执行if语句，即执行

	mDecor = generateDecor();
这里调用[generateDecor()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.generateDecor%28%29)方法获取[DecorView](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.DecorView)对象，那么是如何获取呢，进去瞧一瞧

	protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
很容易理解吧，直接new一个对象，就获取到[DecorView](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.DecorView)实例。再回到[installDecor()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.installDecor%28%29)方法中，对于if语句后面的几行代码，主要是对获取到[DecorView](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.DecorView)对象进行设置。接下来看第7行代码，也是if语句，判断**mContentParent**是否为null，由于一开始进入为null，所以执行if语句，即执行

	mContentParent = generateLayout(mDecor);
这里将在之前获取到的**mDecor**作为参数传递到[generateLayout()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.generateLayout%28com.android.internal.policy.impl.PhoneWindow.DecorView%29)方法中，然后返回ViewGroup赋值给**mContentParent**。那么该方法是如何实现呢，进入该方法代码实现

	protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.
        TypedArray a = getWindowStyle();
        if (false) {
            System.out.println("From style:");
            String s = "Attrs:";
            for (int i = 0; i < com.android.internal.R.styleable.Window.length; i++) {
                s = s + " " + Integer.toHexString(com.android.internal.R.styleable.Window[i]) + "=" +  a.getString(i);
            }
            System.out.println(s);
        }
        mIsFloating = a.getBoolean(com.android.internal.R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(com.android.internal.R.styleable.Window_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestFeature(FEATURE_ACTION_BAR);
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_windowActionBarOverlay, false)) {
            requestFeature(FEATURE_ACTION_BAR_OVERLAY);
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_windowActionModeOverlay, false)) {
            requestFeature(FEATURE_ACTION_MODE_OVERLAY);
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_windowFullscreen, false)) {
            setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN&(~getForcedWindowFlags()));
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_windowShowWallpaper, false)) {
            setFlags(FLAG_SHOW_WALLPAPER, FLAG_SHOW_WALLPAPER&(~getForcedWindowFlags()));
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_windowEnableSplitTouch,
                getContext().getApplicationInfo().targetSdkVersion
                        >= android.os.Build.VERSION_CODES.HONEYCOMB)) {
            setFlags(FLAG_SPLIT_TOUCH, FLAG_SPLIT_TOUCH&(~getForcedWindowFlags()));
        }
        a.getValue(com.android.internal.R.styleable.Window_windowMinWidthMajor, mMinWidthMajor);
        a.getValue(com.android.internal.R.styleable.Window_windowMinWidthMinor, mMinWidthMinor);
        if (getContext().getApplicationInfo().targetSdkVersion
                < android.os.Build.VERSION_CODES.HONEYCOMB) {
            addFlags(WindowManager.LayoutParams.FLAG_NEEDS_MENU_KEY);
        }
        
        if (mAlwaysReadCloseOnTouchAttr || getContext().getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.HONEYCOMB) {
            if (a.getBoolean(
                    com.android.internal.R.styleable.Window_windowCloseOnTouchOutside,
                    false)) {
                setCloseOnTouchOutsideIfNotSet(true);
            }
        }
        
        WindowManager.LayoutParams params = getAttributes();
        if (!hasSoftInputMode()) {
            params.softInputMode = a.getInt(
                    com.android.internal.R.styleable.Window_windowSoftInputMode,
                    params.softInputMode);
        }
        if (a.getBoolean(com.android.internal.R.styleable.Window_backgroundDimEnabled,
                mIsFloating)) {
            /* All dialogs should have the window dimmed */
            if ((getForcedWindowFlags()&WindowManager.LayoutParams.FLAG_DIM_BEHIND) == 0) {
                params.flags |= WindowManager.LayoutParams.FLAG_DIM_BEHIND;
            }
            params.dimAmount = a.getFloat(
                    android.R.styleable.Window_backgroundDimAmount, 0.5f);
        }
        if (params.windowAnimations == 0) {
            params.windowAnimations = a.getResourceId(
                    com.android.internal.R.styleable.Window_windowAnimationStyle, 0);
        }
        // The rest are only done if this window is not embedded; otherwise,
        // the values are inherited from our container.
        if (getContainer() == null) {
            if (mBackgroundDrawable == null) {
                if (mBackgroundResource == 0) {
                    mBackgroundResource = a.getResourceId(
                            com.android.internal.R.styleable.Window_windowBackground, 0);
                }
                if (mFrameResource == 0) {
                    mFrameResource = a.getResourceId(com.android.internal.R.styleable.Window_windowFrame, 0);
                }
                if (false) {
                    System.out.println("Background: " + Integer.toHexString(mBackgroundResource) + " Frame: " + Integer.toHexString(mFrameResource));
                }
            }
            mTextColor = a.getColor(com.android.internal.R.styleable.Window_textColor, 0xFF000000);
        }
        // Inflate the window decor.
        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = com.android.internal.R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = com.android.internal.R.layout.screen_progress;
            // System.out.println("Progress!");
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = com.android.internal.R.layout.screen_custom_title;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        com.android.internal.R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                if ((features & (1 << FEATURE_ACTION_BAR_OVERLAY)) != 0) {
                    layoutResource = com.android.internal.R.layout.screen_action_bar_overlay;
                } else {
                    layoutResource = com.android.internal.R.layout.screen_action_bar;
                }
            } else {
                layoutResource = com.android.internal.R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = com.android.internal.R.layout.screen_simple;
            // System.out.println("Simple!");
        }
        mDecor.startChanging();
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }
        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        if (getContainer() == null) {
            Drawable drawable = mBackgroundDrawable;
            if (mBackgroundResource != 0) {
                drawable = getContext().getResources().getDrawable(mBackgroundResource);
            }
            mDecor.setWindowBackground(drawable);
            drawable = null;
            if (mFrameResource != 0) {
                drawable = getContext().getResources().getDrawable(mFrameResource);
            }
            mDecor.setWindowFrame(drawable);
            // System.out.println("Text=" + Integer.toHexString(mTextColor) +
            // " Sel=" + Integer.toHexString(mTextSelectedColor) +
            // " Title=" + Integer.toHexString(mTitleColor));
            if (mTitleColor == 0) {
                mTitleColor = mTextColor;
            }
            if (mTitle != null) {
                setTitle(mTitle);
            }
            setTitleColor(mTitleColor);
        }
        mDecor.finishChanging();
        return contentParent;
    }
这个方法真的很长，我根据自己的理解总结下这个方法的主要功能：
- 设置窗口属性
- 设置对应布局文件layoutResource
- 加载布局文件layoutResource
- 将加载获取到View添加到**mDecor**
- 设置窗口背景

由于个人能力有限，所以只能简单地分析下。定位到代码

	View in = mLayoutInflater.inflate(layoutResource, null);
根据之前各种条件获取到对应布局文件id，即layoutResource，再调用LayoutInflater.inflate加载布局获取到对应View（如果想理解LayoutInflater原理，可以参考这篇博客[Android LayoutInflater原理分析，带你一步步深入了解View(一)](http://blog.csdn.net/sinyu890807/article/details/12921889)）。接着再看

	decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
把上一步加载布局文件获取到的View添加到**mDecor**，接着

	 ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
通过findViewById**(ID_ANDROID_CONTENT = com.android.internal.R.id.content)**获取到contentParent，最终将**contentParent**作为返回值，赋值给**mContentParent**（记得**id**），这样就获取到**mContentParent**。再回到[installDecor()](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.installDecor%28%29)方法中，剩下的逻辑就自己看吧。再回到[setContentView(int layoutResId)](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.setContentView%28int%29)，有一行关键的代码

	mLayoutInflater.inflate(layoutResID, mContentParent);
这代码挺熟悉吧，这行代码说明加载XML布局到当前Activity还有一个父布局，那就是**mContentParent**，它是一个FrameLayout布局，资源id为**R.id.content**。到此也就了解了XML布局是怎样加载到当前Activity的流程。

