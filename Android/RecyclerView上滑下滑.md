判断RecyclerView滑动完是上滑还是下滑

```java
 scaleRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            private int distanceY = 0;

            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                if (newState == RecyclerView.SCROLL_STATE_DRAGGING) {
                    distanceY = 0;
                } else if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                    if (distanceY > 0) {
                        // 往下滑
                        presenter.preLoadNext();
                    } else if (distanceY < 0) {
                        // 往上滑
                    }
                }
            }

            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                distanceY += dy;
            }
```

