---
layout: post
title: "Android中触摸事件分发机制(3):ViewGroup篇"
date: 2016-06-04 21:52:04 +0800
comments: true
categories: android_deep_analysis
---
在上一篇博客中，我们讲解了View的事件分发过程，这稿博客将分析ViewGroup的事件分发过程，并且在最后给出相应的流程图。  

在"Android触摸事件分发机制(一):Activity篇"的最后，我们说到ViewGroup的触摸事件分发是从dispatchTouchEvent()方法开始的，下面我们就看一下这个方法<!--more-->:  

``` java dispatchTouchEvent()
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
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
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
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

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
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
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                        }
                        if (preorderedList != null) preorderedList.clear();
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
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
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
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

```

由于ViewGroup继承自View,而View中也有dispatchTouchEvent()方法，所以ViewGroup中的dispatchTouchEvent()是覆写了View中的dispatchTouchEvent()，也说明了它们对触摸事件的处理有所不同。

为了详细地了解整个触摸事件传递过程，这里从ACTION_DOWN开始进行分析。  

首先是onFilterTouchEventForSecurity()，这个在View的事件分发中已经讲过，就不再赘述;还有由于MotionEvent.ACTION_DOWN是触摸事件的开始，所以如果是ACTION_DOWN事件的话就需要重置状态。开始时是ACTION_DOWN状态，所以进入cancelAndClearTouchTargets()中，下面是该方法：  

``` java cancelAndClearTouchTargets()
   /**
     * Cancels and clears all touch targets.
     */
    private void cancelAndClearTouchTargets(MotionEvent event) {
        if (mFirstTouchTarget != null) {
            boolean syntheticEvent = false;
            if (event == null) {
                final long now = SystemClock.uptimeMillis();
                event = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
                syntheticEvent = true;
            }

            for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
                resetCancelNextUpFlag(target.child);
                dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
            }
            clearTouchTargets();

            if (syntheticEvent) {
                event.recycle();
            }
        }
    }
```
resetCancelNextUpFlag()和dispatchTransformedTouchEvent()到后面再讲，这里暂时不影响我们理解，就先看clearTouchTargets();该方法如下：  

``` java clearTouchTargets()
 /**
     * Clears all touch targets.
     */
    private void clearTouchTargets() {
        TouchTarget target = mFirstTouchTarget;
        if (target != null) {
            do {
                TouchTarget next = target.next;
                target.recycle();
                target = next;
            } while (target != null);
            mFirstTouchTarget = null;
        }
    }

```
显然，会将所有的TouchTarget都回收，并且使mFirstTouchTarget为null.

回到dispatchTouchEvent()方法中，虽然此时mFirstTouchTarget为null,但是actionMasked==MotionEvent.ACTION_DOWN成立，所以会进入到第一个分支中。
下面首先看mGroupFlags的值。我们看一下它在哪里被赋值，找了一下，发现是在requestDisallowInterceptTouchEvent()方法中，如下所示：
``` java requestDisallowInterceptTouchEvent()
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

        if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
            // We're already in this state, assume our ancestors are too
            return;
        }

        if (disallowIntercept) {
            mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
        } else {
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // Pass it up to our parent
        if (mParent != null) {
            mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
        }
    }
```
显然，如果disallowIntercept为true的话，mGroupFlags的相应位就与FLAG_DISALLOW_INTERCEPT一致,否则与FLAG_DISALLOW_INTERCEPT相反。假设在ViewGroup的外部调用了requestDisallowInterceptTouchEvent(true);则这里intercepted始终为false,否则进入到onInterceptTouchEvent()中，该方法如下：  

``` java onInterceptTouchEvent()
public boolean onInterceptTouchEvent(MotionEvent ev){
	return false;
}
```
这个方法异常的简单，即默认情况下返回false,如果用户重写了该方法，比如返回true，则ViewGroup会截取之后的事件，在ViewGroup中的子View就接收不到后面的事件了。  

还有一个分支就是如果不是ACTION_DOWN事件，并且mFirstTouchTarget==null,则表示没有相应的事件接收目标，自然也就没必要再往下传递事件了，所以此时intercept=true;

接着往下看，是确认事件是否被取消，如果是的话就不会进行后续的处理。  

然后是获取一个boolean变量split来标记是否把事件分发给多个子View，它的赋值其实跟上面的requestDisallowInterceptTouchEvent()类似，是在setMotionEventSplittingEnabled()中，该方法如下所示：  

``` java setMotionEventSplittingEnabled()
    /**
     * Enable or disable the splitting of MotionEvents to multiple children during touch event
     * dispatch. This behavior is enabled by default for applications that target an
     * SDK version of {@link Build.VERSION_CODES#HONEYCOMB} or newer.
     *
     * <p>When this option is enabled MotionEvents may be split and dispatched to different child
     * views depending on where each pointer initially went down. This allows for user interactions
     * such as scrolling two panes of content independently, chording of buttons, and performing
     * independent gestures on different pieces of content.
     *
     * @param split <code>true</code> to allow MotionEvents to be split and dispatched to multiple
     *              child views. <code>false</code> to only allow one child view to be the target of
     *              any MotionEvent received by this ViewGroup.
     * @attr ref android.R.styleable#ViewGroup_splitMotionEvents
     */
    public void setMotionEventSplittingEnabled(boolean split) {
        // TODO Applications really shouldn't change this setting mid-touch event,
        // but perhaps this should handle that case and send ACTION_CANCELs to any child views
        // with gestures in progress when this is changed.
        if (split) {
            mGroupFlags |= FLAG_SPLIT_MOTION_EVENTS;
        } else {
            mGroupFlags &= ~FLAG_SPLIT_MOTION_EVENTS;
        }
    }

```
如果事件既没有被取消，也没有被当前ViewGroup截取，那就会进入下面复杂的处理了。  

由于现在是ACTION_DOWN事件，那么actionIndex为0，后面那些idBitsToAssign和removePointersFromTouchTargets()不影响我们理解，暂时先不讲。  

现在newTouchTarget为null,如果ViewGroup中有子View的话，则进入该分支中。  

首先是获取到x和y坐标，然后是通过前序遍历的方法来构建所有的子View，之所以采用前序遍历，是因为preorderedList中的顺序是按照addView或者XML布局文件中的顺序而来的，后addView添加的子View，会因为Android的UI后刷新机制显示在上层。假如点击的地方有两个子View都包含在其中，那么后添加进去的那个子View会先响应事件，这样其实也是符合人的思维方式的，因为后被添加的子View会在图层的上层，自然我们点击时也是希望上层的View最先响应。  

结合上面的分析，就能理解for循环采用倒序遍历的原因了。之后通过canViewReceivePointerEvents()和isTransformedTouchPointInVieww()方法来判断当前child是否能够接受事件，并且点击位置是否在child的范围内，如果不满足则continue.

一直遍历到找到第一个符合上述条件的child为止，找到之后利用进入getTouchTarget()方法中，如下所示：  

``` java getTouchTarget()
    /**
     * Gets the touch target for specified child view.
     * Returns null if not found.
     */
    private TouchTarget getTouchTarget(View child) {
        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            if (target.child == child) {
                return target;
            }
        }
        return null;
    }

```
显然，getTouchTarget()是遍历以mFirstTouchTarget为首的TouchTarget链条,看child是否存在某个TouchTarget中。  

前面分析过，此时为ACTION_DOWN事件，所以mFirstTouchTarget为null,从而getTouchTarget()返回null.所以暂时不会break,而是执行到dispatchTransformedTouchEvent()处，这个方法非常重要。下面是该方法体：  

``` java dispatchTransformedTouchEvent()
    /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
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
首先判断是否要取消事件，如果要取消的话则进行判断，如果child为null则执行super.dispatchTouchEvent()，即当前ViewGroup作为View的dispatchTouchEvent()方法，关于View的dispatchTouchEvent()方法，在上一篇博客中详细讲过，此处不再赘述;如果child不为null,则调用child.dispatchTouchEvent()，注意这里其实就进入递归了，因为dispatchTransformedTouchEvent()是在ViewGroup#dispatchTouchEvent()中被调用,而这里又调用child.dispatchTouchEvent()方法，如果child也是ViewGroup，则进入ViewGroup的dispatchTouchEvent()，否则调用的是View的dispatchTouchEvent()方法。

如果不取消事件或者不是ACTION_CANCEL事件，则分两种情况，一种是newPointerIdBits==oldPointerIdBits,此时说明当前的事件和之前的事件相同，因而不需要变形，可直接调用super.dispatchTouchEvent(event)或者child.dispatchTouchEvent(event),否则需要调用MotionEvent.obtain(event)获得transformedEvent,之后对变形后的事件进行类似的处理。 

如果事件被child或其子View消费掉，则dispatchTransformedTouchEvent()会返回true,从而进入下面的分支：先获取到mLastTouchDownTime,mLastTouchDownIndex和mLastTouchDownX,mLastTouchDownY,之后通过addTouchTarget()得到一个TouchTarget对象，下面是该方法：  

``` java addTouchTarget()
   /**
     * Adds a touch target for specified child to the beginning of the list.
     * Assumes the target child is not already present.
     */
    private TouchTarget addTouchTarget(View child, int pointerIdBits) {
        TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }

```
之后跳出循环，后面的处理其实主要是针对非ACTION_DOWN的情况，可以看到后续还有dispatchTransformedTouchEvent()方法的调用，即为非ACTION_DOWN情况下也进行类似的递归调用，只要有一个child消费了该事件，就会使handled=true;从而返回true.

非ACTION_DOWN情形要简单得多，主要就是160行之后的处理，上面已经分析过，就不再赘述了。

至此，我们可以得到ViewGroup的触摸事件分发机制的流程图：  

![ViewGroup_dispatchTouchEvent](http://7xn1yt.com1.z0.glb.clouddn.com/ViewGroup_dispatchTouchEvent.png)


