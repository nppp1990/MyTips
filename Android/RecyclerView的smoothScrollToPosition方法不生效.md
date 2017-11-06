# RecyclerView的smoothScrollToPosition方法不生效

[TOC]

## 前言

项目中要滑动到列表页某个指定位置（RecyclerView+LinearLayoutManager）,于是就想到用smoothScrollToPosition，但是有时候滑不到具体的位置

## 问题原因

LinearLayoutManager的scrollToPosition和smoothScrollToPosition效果是这样的

1. .如果position的item没有在完全展示在屏幕的列表内，可以正确的展示我们要的效果（item的上沿展示在列表的最上面）
2. 如果item已经完全展示了，则不会滑动到指定的位置

## 解决

那怎么解决呢，可以用scrollToPositionWithOffset

```java
        LinearLayoutManager layoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();
        int last = layoutManager.findLastVisibleItemPosition();
        int first = layoutManager.findFirstVisibleItemPosition();
        if (position > first && position <= last) {
            layoutManager.scrollToPositionWithOffset(position, 0);//这种滑动没有动画效果，会有点突兀
        } else {
            recyclerView.smoothScrollToPosition(position);
        }
```

