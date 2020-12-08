
# Android RecyclerView 缓存机制

  最近学习了一下RecyclerView 缓存机制的缓存机制，为了防止忘了，记录下相关的知识点。

## 一、RecyclerView四级缓存

  RecyclerView使用四级缓存来进行视图的复用，缓存的逻辑主要集中在内部类Recycler中，下面是这个类的成员：

```
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;

    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

    private final List<ViewHolder>
            mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);

    private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
    int mViewCacheMax = DEFAULT_CACHE_SIZE;

    RecycledViewPool mRecyclerPool;

    private ViewCacheExtension mViewCacheExtension;

    static final int DEFAULT_CACHE_SIZE = 2;
}
  从这个类的成员可以看到，RecyclerView的四级缓存就是以下的4个：

1. mAttachedScrap--用于快速重用屏幕上可见的列表项ItemView，而不需要重新createView和bindView
2. mCachedViews--缓存离开屏幕的ItemView，目的是让即将进入屏幕的ItemView重用，使用这个缓存时不需要重新createView和bindView
3. mViewCacheExtension--不直接使用，用户定制的缓存
4. mRecyclerPool--也是缓存离开屏幕的ItemView，但需要重新bindView

## 二、源码分析—复用机制

  接下来看下这四级缓存是怎么工作的，如何复用ItemView。下面直接看源码

```
/**
 * Attempts to get the ViewHolder for the given position, either from the Recycler scrap,
 * cache, the RecycledViewPool, or creating it directly.
 **/
    /**
     * 注释写的很清楚，从Recycler的scrap，cache，RecyclerViewPool,或者直接create创建
     **/
    @Nullable
    ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                                                     boolean dryRun, long deadlineNs) {
        if (position < 0 || position >= mState.getItemCount()) {
            throw new IndexOutOfBoundsException("Invalid item position " + position
                    + "(" + position + "). Item count:" + mState.getItemCount()
                    + exceptionLabel());
        }
        boolean fromScrapOrHiddenOrCache = false;
        ViewHolder holder = null;
        // 0) If there is a changed scrap, try to find from there
        if (mState.isPreLayout()) {
            //preLayout默认是false，只有有动画的时候才为true
            holder = getChangedScrapViewForPosition(position);
            fromScrapOrHiddenOrCache = holder != null;
        }
        // 1) Find by position from scrap/hidden list/cache
        if (holder == null) {
            holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
            if (holder != null) {
                if (!validateViewHolderForOffsetPosition(holder)) {
                    //如果检查发现这个holder不是当前position的
                    // recycle holder (and unscrap if relevant) since it can't be used
                    if (!dryRun) {
                        // we would like to recycle this but need to make sure it is not used by
                        // animation logic etc.
                        holder.addFlags(ViewHolder.FLAG_INVALID);
                        //从scrap中移除
                        if (holder.isScrap()) {
                            removeDetachedView(holder.itemView, false);
                            holder.unScrap();
                        } else if (holder.wasReturnedFromScrap()) {
                            holder.clearReturnedFromScrapFlag();
                        }
                        //放到ViewCache或者Pool中
                        recycleViewHolderInternal(holder);
                    }
                    //至空继续寻找
                    holder = null;
                } else {
                    fromScrapOrHiddenOrCache = true;
                }
            }
        }
        if (holder == null) {
            final int offsetPosition = mAdapterHelper.findPositionOffset(position);
            if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
                throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                        + "position " + position + "(offset:" + offsetPosition + ")."
                        + "state:" + mState.getItemCount() + exceptionLabel());
            }

            final int type = mAdapter.getItemViewType(offsetPosition);
            // 2) Find from scrap/cache via stable ids, if exists
            if (mAdapter.hasStableIds()) {
                holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                        type, dryRun);
                if (holder != null) {
                    // update position
                    holder.mPosition = offsetPosition;
                    fromScrapOrHiddenOrCache = true;
                }
            }
            //自定义缓存
            if (holder == null && mViewCacheExtension != null) {
                // We are NOT sending the offsetPosition because LayoutManager does not
                // know it.
                final View view = mViewCacheExtension
                        .getViewForPositionAndType(this, position, type);
                if (view != null) {
                    holder = getChildViewHolder(view);
                    if (holder == null) {
                        throw new IllegalArgumentException("getViewForPositionAndType returned"
                                + " a view which does not have a ViewHolder"
                                + exceptionLabel());
                    } else if (holder.shouldIgnore()) {
                        throw new IllegalArgumentException("getViewForPositionAndType returned"
                                + " a view that is ignored. You must call stopIgnoring before"
                                + " returning this view." + exceptionLabel());
                    }
                }
            }
            //pool
            if (holder == null) { // fallback to pool
                if (DEBUG) {
                    Log.d(TAG, "tryGetViewHolderForPositionByDeadline("
                            + position + ") fetching from shared pool");
                }
                holder = getRecycledViewPool().getRecycledView(type);
                if (holder != null) {
                    holder.resetInternal();
                    if (FORCE_INVALIDATE_DISPLAY_LIST) {
                        invalidateDisplayListInt(holder);
                    }
                }
            }
            //create
            if (holder == null) {
                long start = getNanoTime();
                if (deadlineNs != FOREVER_NS
                        && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                    // abort - we have a deadline we can't meet
                    return null;
                }
                holder = mAdapter.createViewHolder(RecyclerView.this, type);
                if (ALLOW_THREAD_GAP_WORK) {
                    // only bother finding nested RV if prefetching
                    RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                    if (innerView != null) {
                        holder.mNestedRecyclerView = new WeakReference<>(innerView);
                    }
                }

                long end = getNanoTime();
                mRecyclerPool.factorInCreateTime(type, end - start);
            }
        }
        ....
        return holder;
    }
```
  ecycler复用机制主要在tryGetViewHolderForPositionByDeadline进行，会从从Recycler的scrap，cache，RecyclerViewPool创建，如果都找不到则createView

### 1、第一次尝试（从mAttachedScrap和mCachedViews获取）

```
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
        final int scrapCount = mAttachedScrap.size();

        // Try first for an exact, non-invalid match from scrap.
        //先从scrap中寻找
        for (int i = 0; i < scrapCount; i++) {
            final ViewHolder holder = mAttachedScrap.get(i);
            return holder;
           ...
        }
        //dryRun为false
        if (!dryRun) {
            //从HiddenView中获得，这里获得是View
            View view = mChildHelper.findHiddenNonRemovedView(position);
            if (view != null) {
                // This View is good to be used. We just need to unhide, detach and move to the
                // scrap list.
                //通过View的LayoutParam获得ViewHolder
                final ViewHolder vh = getChildViewHolderInt(view);
                //从HiddenView中移除
                mChildHelper.unhide(view);
                ....
                mChildHelper.detachViewFromParent(layoutIndex);
                //添加到Scrap中，其实这里既然已经拿到了ViewHolder，可以直接传vh进去
                scrapView(view);
                vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                        | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                return vh;
            }
        }

        // Search in our first-level recycled view cache.
        //从CacheView中拿
        final int cacheSize = mCachedViews.size();
        for (int i = 0; i < cacheSize; i++) {
            final ViewHolder holder = mCachedViews.get(i);
            // invalid view holders may be in cache if adapter has stable ids as they can be
            // retrieved via getScrapOrCachedViewForId
            //holder是有效的，并且position相同
            if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
                if (!dryRun) {
                    mCachedViews.remove(i);
                }
                return holder;
            }
        }
        return null;
    }

```

  第一次尝试调用getScrapOrHiddenOrCachedHolderForPosition，通过position从mAttachedScrap，mHiddenViews和mCachedViews中获取

### 2、第二次尝试

```
ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
        // Look in our attached views first
        //
        final int count = mAttachedScrap.size();
        for (int i = count - 1; i >= 0; i--) {
            //在attachedScrap中寻找
            final ViewHolder holder = mAttachedScrap.get(i);
            if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
                //id相同并且不是从scrap中返回的
                if (type == holder.getItemViewType()) {
                    holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                    if (holder.isRemoved()) {
                        // this might be valid in two cases:
                        // > item is removed but we are in pre-layout pass
                        // >> do nothing. return as is. make sure we don't rebind
                        // > item is removed then added to another position and we are in
                        // post layout.
                        // >> remove removed and invalid flags, add update flag to rebind
                        // because item was invisible to us and we don't know what happened in
                        // between.
                        if (!mState.isPreLayout()) {
                            holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE
                                    | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);
                        }
                    }
                    return holder;
                } else if (!dryRun) {
                    // if we are running animations, it is actually better to keep it in scrap
                    // but this would force layout manager to lay it out which would be bad.
                    // Recycle this scrap. Type mismatch.
                    //从scrap中移除
                    mAttachedScrap.remove(i);
                    removeDetachedView(holder.itemView, false);
                    //加入cacheView或者pool
                    quickRecycleScrapView(holder.itemView);
                }
            }
        }
        //从cacheView中找
        // Search the first-level cache
        final int cacheSize = mCachedViews.size();
        for (int i = cacheSize - 1; i >= 0; i--) {
            final ViewHolder holder = mCachedViews.get(i);
            if (holder.getItemId() == id) {
                if (type == holder.getItemViewType()) {
                    if (!dryRun) {
                        //从cache中移除
                        mCachedViews.remove(i);
                    }
                    return holder;
                } else if (!dryRun) {
                    //从cacheView中移除，但是放到pool中
                    recycleCachedViewAt(i);
                    return null;
                }
            }
        }
        return null;
    }

  第二次尝试调用getScrapOrCachedViewForId，通过id来寻找，当我们将hasStableIds()设为true后就会进行查找

### 3、第三次尝试(对应于自定义缓存)

```
if (holder == null && mViewCacheExtension != null) {
                // We are NOT sending the offsetPosition because LayoutManager does not
                // know it.
                final View view = mViewCacheExtension
                        .getViewForPositionAndType(this, position, type);
                if (view != null) {
                    holder = getChildViewHolder(view);
                    if (holder == null) {
                        throw new IllegalArgumentException("getViewForPositionAndType returned"
                                + " a view which does not have a ViewHolder"
                                + exceptionLabel());
                    } else if (holder.shouldIgnore()) {
                        throw new IllegalArgumentException("getViewForPositionAndType returned"
                                + " a view that is ignored. You must call stopIgnoring before"
                                + " returning this view." + exceptionLabel());
                    }
                }
            }
```

  自定义缓存可以让用户自行设置缓存的策略，这里就不多说了

### 4、第四次尝试（mRecyclerPool）

```
public static class  {
    private static final int DEFAULT_MAX_SCRAP = 5;
    static class ScrapData {
        ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
        long mCreateRunningAverageNs = 0;
        long mBindRunningAverageNs = 0;
    }
    SparseArray<ScrapData> mScrap = new SparseArray<>();

    private int mAttachCount = 0;
    ...
    public ViewHolder getRecycledView(int viewType) {
        final ScrapData scrapData = mScrap.get(viewType);
        if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
            final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
            return scrapHeap.remove(scrapHeap.size() - 1);
        }
        return null;
    }
    ......
  }
```

  当从上面的缓存找不到的时候，就会在RecycledViewPool中寻找，里面使用了来存放对应viewtype的ViewHolder，因为从RecycledViewPool中寻找并不会根据对应的position，而是有则返回，因此需要重新进行bindView.

### 5、创建CreateView

```
if (holder == null) {
    long start = getNanoTime();
    if (deadlineNs != FOREVER_NS
            && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
        // abort - we have a deadline we can't meet
        return null;
    }
    holder = mAdapter.createViewHolder(RecyclerView.this, type);
}
```

  当所有的缓存都找不到，则进行createViewHolder

## 三、源码分析—回收机制

  上面看了Recycler是如何从读取缓存的，那现在就看一下是如何进行回收的

```
void recycleViewHolderInternal(ViewHolder holder) {
    // 省略一些前置判断
    //noinspection unchecked
    final boolean transientStatePreventsRecycling = holder
            .doesTransientStatePreventRecycling();
    final boolean forceRecycle = mAdapter != null
            && transientStatePreventsRecycling
            && mAdapter.onFailedToRecycleView(holder);
    boolean cached = false;
    boolean recycled = false;
    if (DEBUG && mCachedViews.contains(holder)) {
        throw new IllegalArgumentException("cached view received recycle internal? "
                + holder + exceptionLabel());
    }
    if (forceRecycle || holder.isRecyclable()) {
        if (mViewCacheMax > 0
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
            // Retire oldest cached view
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                recycleCachedViewAt(0);
                cachedViewSize--;
            }

            int targetCacheIndex = cachedViewSize;
            if (ALLOW_THREAD_GAP_WORK
                    && cachedViewSize > 0
                    && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                // when adding the view, skip past most recently prefetched views
                int cacheIndex = cachedViewSize - 1;
                while (cacheIndex >= 0) {
                    int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                    if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                        break;
                    }
                    cacheIndex--;
                }
                targetCacheIndex = cacheIndex + 1;
            }
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    }
    // even if the holder is not removed, we still call this method so that it is removed
    // from view holder lists.
    mViewInfoStore.removeViewHolder(holder);
    if (!cached && !recycled && transientStatePreventsRecycling) {
        holder.mOwnerRecyclerView = null;
    }
}
```
  Recycler的回收机制主要集中在recycleViewHolderInternal，当视图移除屏幕，就会将ViewHolder放进mCachedViews里，如果mCachedViews满了则会放进RecycledViewPool里

## 四、总结

1. RecyclerView四级缓存：mAttachedScrap，mCachedViews，mViewCacheExtension，mRecyclerPool
2. mCachedViews 优先级高于 RecyclerViewPool，回收时，最新的 ViewHolder 都是往 mCachedViews 里放，如果它满了，那就移出一个扔到 ViewPool 里好空出位置来缓存最新的 ViewHolder。
3. mAttachedScrap，mCachedViews取出来的均不需要重新bindView，而mRecyclerPool取出来的则需要进行bindView
