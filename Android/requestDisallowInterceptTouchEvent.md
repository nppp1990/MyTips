### requestDisallowInterceptTouchEvent

```java
/**
 * Called when a child does not want this parent and its ancestors to
 * intercept touch events with
 * {@link ViewGroup#onInterceptTouchEvent(MotionEvent)}.
 *
 * <p>This parent should pass this call onto its parents. This parent must obey
 * this request for the duration of the touch (that is, only clear the flag
 * after this parent has received an up or a cancel.</p>
 * 
 * @param disallowIntercept True if the child does not want the parent to
 *            intercept touch events.
 */
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept);
```

case：横向的RecyclerView嵌套一个横向的滚动的view，由于RecyclerView拦截了滚动的事件，导致子view不能正常的横向滚动

solution：

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent e) {
    if (parent != null) {
        if (e.getAction() == MotionEvent.ACTION_MOVE) {
            parent.requestDisallowInterceptTouchEvent(true);
        } else {
            parent.requestDisallowInterceptTouchEvent(false);
        }
    }
    return super.onInterceptTouchEvent(e);
}
```

