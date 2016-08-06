---
layout: post
title: "Android中触摸事件分发机制(2):View篇"
date: 2016-06-04 21:47:00 +0800
comments: true
categories: android_deep_analysis
---
在上一篇文章的末尾说到接下来要分析ViewGroup的事件分发过程。但是考虑到ViewGroup继承自View,它们有一些共同的事件分发过程，所以如果先讲解View的事件分发过程的话，会有助于对于ViewGroup的理解和对比。  

View的事件分发是从dispatchTouchEvent方法开始的(在下一篇分析ViewGroup的事件分发过程中会讲到为什么是从这个方法开始的)，该方法如下<!--more-->：  

``` java dispatchTouchEvent()
 /**
     * Pass the touch screen motion event down to the target view, or this
     * view if it is the target.
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     */
    public boolean dispatchTouchEvent(MotionEvent event) {
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
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
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
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }

```
其中mInputEventConsistencyVerifier是跟输入事件相关的，这里先不管这个。
而如果当前事件为ACTION_DOWN的话，则清除之前的手势以开始新的手势判断。  
下面的onFilterTouchEventForSecurity()方法如下：  
``` java onFilterTouchEventForSecurity()
/**
     * Filter the touch event to apply security policies.
     *
     * @param event The motion event to be filtered.
     * @return True if the event should be dispatched, false if the event should be dropped.
     *
     * @see #getFilterTouchesWhenObscured
     */
    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // Window is obscured, drop this touch.
            return false;
        }
        return true;
    }
```
显然，它的作用就是判断当前View是否被遮蔽，如果是的话则返回false,事件被丢弃;否则返回true.  
如果当前View未被遮蔽，则会进行进一步的判断，主要是涉及enable属性以及OnTouchListener,如果mViewFlags&ENABLED_MASK==ENABLED并且mOnTouchListener不为null的话，则会调用mOnTouchListener的onTouch()方法，这个mOnTouchListener就是我们为View设置的OnTouchListener，并且注意执行之后返回true,返回true表示在这里将事件消费了，从而在ViewGroup的ViewTree中该触摸事件不再往后传递了。  

由此我们可得到结论一：如果当前View的enable对应位为ENABLED并且为其设置了OnTouchListener,并且mOnTouchListener.onTouch()的返回值为true,那么触摸事件会在此被消费，并且不会再往下传递，而且View的onTouchEvent()方法也不会被调用。

继续往下分析，即如果enable对应位不为ENABLED或者未设置OnTouchListener，则result仍然为false，从而会执行View的onTouchEvent()方法，如果onTouchEvent()方法的返回值为true的话，则result为true,表示此次触摸事件在当前View中被消费，不再往后传递了。所以由此可得到结论二。

结论二：如果View的mViewFlags属性不满足条件，或者其OnTouchListener为null,则不会执行OnTouchListener的onTouch()方法，而是会执行View自身的onTouchEvent()方法，并且将这次事件消费掉。

下面我们就看一下onTouchEvent()方法：  

``` java onTouchEvent()
 /**
     * Implement this method to handle touch screen motion events.
     * <p>
     * If this method is used to detect click actions, it is recommended that
     * the actions be performed by implementing and calling
     * {@link #performClick()}. This will ensure consistent system behavior,
     * including:
     * <ul>
     * <li>obeying click sound preferences
     * <li>dispatching OnClickListener calls
     * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
     * accessibility features are enabled
     * </ul>
     *
     * @param event The motion event.
     * @return True if the event was handled, false otherwise.
     */
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
        }

        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    break;

                case MotionEvent.ACTION_MOVE:
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
    }

```
可以看到这个方法很长，但是其实重点内容也不多，主要是以下3部分：

+ (1)如果View被禁用的话，则返回它是否可以被点击;

+ (2)如果该View的mTouchDelegate不为null的话，则将触摸事件分发给mTouchDelegate，并且在这里将事件消费掉，不再让事件向后传递。关于TouchDelegate，只要了解这点：假如有两个View对象v1和v2,它们之间不重叠，将v2设置为v1的mTouchDelegate的话，则v1的触摸事件就会被分发给v2.注意默认状态下mTouchDelegate为null;
+ (3)如果View可以被点击的话，则针对不同的动作进行相应的处理。这里主要涉及的几个方法是removeLongPressCallback(),removeTapCallback(),setPressed()和performClick()方法。  

其中removeLongPressCallback()和removeTapCallback()都很好理解，就是在相应的动作中要移除长按对应的runnable和tap对应的runnable，以便开始新的手势判断。而setPressed()则是设置按压时的状态，比如Button按压时变色。而performClick()是大家非常熟悉的方法了，即执行点击事件。  

到这里，我们就可以画出View的事件分发过程流程图了：  

![view_dispatchTouchEvent](http://7xn1yt.com1.z0.glb.clouddn.com/view_dispatchTouchEvent.png)

下一篇文章我们就将分析ViewGroup的事件分发机制。
