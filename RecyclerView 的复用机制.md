> RecyclerView 是一个强大又灵活的 View，可以用有限的 View 来展示大量的数据。今天我们来看下 RecyclerView 内部是通过怎样的缓存复用机制来实现这一功能的。

## Recycler

Recycler 是 RecyclerView 的内部类，也是这套复用机制的核心，显然 Recycler 的主要成员变量也都是用来缓存和复用 ViewHolder 的：

```JAVA
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;

    final ArrayList<ViewHolder> mCachedViews = new ArrayList<>();
    
    RecycledViewPool mRecyclerPool;
    
    private ViewCacheExtension mViewCacheExtension;
}
```

这些缓存集合可以分为 4 个级别，按优先级从高到底为：

- 一级缓存：mAttachedScrap 和 mChangedScrap ，用来缓存还在屏幕内的 ViewHolder

  - mAttachedScrap 存储的是当前还在屏幕中的 ViewHolder；按照 id 和 position 来查找 ViewHolder
  - mChangedScrap 表示数据已经改变的 ViewHolder 列表, 存储 notifyXXX 方法时需要改变的 ViewHolder

- 二级缓存：mCachedViews ，用来缓存移除屏幕之外的 ViewHolder，默认情况下缓存容量是 2，可以通过 setViewCacheSize 方法来改变缓存的容量大小。如果 mCachedViews 的容量已满，则会根据 FIFO 的规则移除旧 ViewHolder 

- 三级缓存：ViewCacheExtension ，开发给用户的自定义扩展缓存，需要用户自己管理 View 的创建和缓存。个人感觉这个拓展脱离了 Adapter.createViewHolder 使用的话会造成 View 创建和数据绑定和其它代码太分散，不利于维护，使用场景很少仅做了解

  ```java
  /*
  	* Note that, Recycler never sends Views to this method to be cached. It is developers
    * responsibility to decide whether they want to keep their Views in this custom cache or let
    * the default recycling policy handle it.
    */
  public abstract static class ViewCacheExtension {
  	public abstract View getViewForPositionAndType(...);
  }
  ```

- 四级缓存：RecycledViewPool ，ViewHolder 缓存池，在有限的 mCachedViews 中如果存不下新的 ViewHolder 时，就会把 ViewHolder 存入 RecyclerViewPool 中。

  - 按照 Type 来查找 ViewHolder
  - 每个 Type 默认最多缓存 5 个
  - 可以多个 RecyclerView 共享 RecycledViewPool

接下来我们看下这四级缓存是怎么工作的

## 复用

RecyclerView 作为一个 “平平无奇” 的 View，子 View 的排列和布局当然是从 onLayout 入手了，调用链：

```java
   RecyclerView.onLayout(...)
-> RecyclerView.dispatchLayout()    
-> RecyclerView.dispatchLayoutStep2() // do the actual layout of the views for the final state.
-> mLayout.onLayoutChildren(mRecycler, mState) // mLayout 类型为 LayoutManager
-> LinearLayoutManager.onLayoutChildren(...) // 以 LinearLayoutManager 为例
-> LinearLayoutManager.fill(...) // The magic functions :) 填充给定的布局，注释很自信的说这个方法很独立，稍微改动就能作为帮助类的一个公开方法，程序员的快乐就是这么朴实无华。
-> LinearLayoutManager.layoutChunk(recycler, layoutState) // 循环调用，每次调用填充一个 ItemView 到 RV
-> LinearLayoutManager.LayoutState.next(recycler) 
-> RecyclerView.Recycler.getViewForPosition(int) // 回到主角了,通过 Recycler 获取指定位置的 ItemView    
-> Recycler.getViewForPosition(int, boolean) // 调用下面方法获取 ViewHolder,并返回上面需要的 viewHolder.itemView 
-> Recycler.tryGetViewHolderForPositionByDeadline(...) // 终于找到你，还好没放弃~    
```

可以看出最终调用 tryGetViewHolderForPositionByDeadline ，来看看这个方法是怎么拿到相应位置上的 ViewHolder ：

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position, ...) {
    if (mState.isPreLayout()) {
        // 0) 预布局从 mChangedScrap 里面去获取 ViewHolder，动画相关
        holder = getChangedScrapViewForPosition(position);
    }
    
    if (holder == null) {
        // 1) 分别从 mAttachedScrap、 mHiddenViews、mCachedViews 获取 ViewHolder
        //    这个 mHiddenViews 是用来做动画期间的复用
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    
    if (holder == null) {
        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2) 如果 Adapter 的 hasStableIds 方法返回为 true
        //    优先通过 ViewType 和 ItemId 两个条件从 mAttachedScrap 和 mCachedViews 寻找
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
        }
      
        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not know it.
            // 3) 从自定义缓存获取，别问，问就是别用
            View view = mViewCacheExtension
                getViewForPositionAndType(this, position, type);
            holder = getChildViewHolder(view);
        }
    }
    
    if (holder == null) {
        // 4) 从 RecycledViewPool 获取 ViewHolder
        holder = getRecycledViewPool().getRecycledView(type);
    }
  
    if (holder == null) {
        // 缓存全取过了，没有，那只好 create 一个咯
        holder = mAdapter.createViewHolder(RecyclerView.this, type);
    }
    
    // This is very ugly but the only place we can grab this information
    // 大半夜看到这句注释的时候笑出声！像极了我写出丑代码时的无奈。
    
    // 在这后面有一些刷新 ViewHolder 信息的代码，放这很丑，但又只能放这，为了能走到这，前面有多少 if (holder == null)
}
```

分析完复用的部分，接下来再看一下 ViewHolder 存入缓存的部分

## 缓存

所谓的缓存，就是看一下是怎么样往前面提到的四级缓存添加数据的

* mAttachedScrap 和 mChangedScrap 
* mCachedViews
* ~~ViewCacheExtension~~ 前面说了，这个的创建和缓存完全由开发者自己控制，系统未往这里添加数据
* RecycledViewPool

### mAttachedScrap 和 mChangedScrap

如果调用了 Adapter 的 notifyXXX 方法，会重新回调到 LayoutManager 的onLayoutChildren 方法里面, 而在 onLayoutChildren 方法里面，会将屏幕上所有的 ViewHolder 回收到 mAttachedScrap 和 mChangedScrap。

```JAVA
// 调用链 
   LinearLayoutManager.onLayoutChildren(...)
-> LayoutManager.detachAndScrapAttachedViews(recycler)
-> LayoutManager.scrapOrRecycleView(..., view)
-> Recycler.scrapView(view);  

private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        // 后面会讲，缓存到 mCacheViews 和 RecyclerViewPool
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        // 缓存到 scrap
        recycler.scrapView(view); 
    }
}

void scrapView(View view) {
    final ViewHolder holder = getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
        || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
        // 标记为移除或失效的 || 完全没有改变 || item 无动画或动画不复用
        mAttachedScrap.add(holder);
    } else {
        mChangedScrap.add(holder);
    }
}
```

其实还有一种情况会调用到 scrapView , 从 mHiddenViews 获得一个 ViewHolder 的话（发生在支持动画的操作），会先将这个 ViewHolder 从 mHiddenViews 数组里面移除，然后调用:
```java
   Recycler.tryGetViewHolderForPositionByDeadline(...)
-> Recycler.getScrapOrHiddenOrCachedHolderForPosition(...)
-> Recycler.scrapView(view)
```

### mCacheViews 和 RecyclerViewPool

这两级缓存的代码都在 Recycler 的这个方法里：

```java
void recycleViewHolderInternal(ViewHolder holder) {
    if (forceRecycle || holder.isRecyclable()) {
        if(mViewCacheMax > 0
             && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                 | ViewHolder.FLAG_REMOVED
                 | ViewHolder.FLAG_UPDATE
                 | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
          int cachedViewSize = mCachedViews.size();
          if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
              // 1. mCacheViews 满了，最早加入的不要了放 RecyclerViewPool
              recycleCachedViewAt(0); 
          }
           mCachedViews.add(targetCacheIndex, holder);
           cached = true;
        }
        
        if (!cached) { 
            // 2. 不能放进 mCacheViews 的放 RecyclerViewPool
            addViewHolderToRecycledViewPool(holder, true);
        }
    }   
}

// Recycles a cached view and removes the view from the list
void recycleCachedViewAt(int cachedViewIndex) {
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);
    addViewHolderToRecycledViewPool(viewHolder, true);
    mCachedViews.remove(cachedViewIndex);
}
```

在这我们知道 recycleViewHolderInternal 会把 ViewHolder 缓存到 mCacheViews ，而不满足加到 mCacheViews 的会缓存到 RecycledViewPool 。那又是什么时候调用的 recycleViewHolderInternal 呢？有以下三种情况：

1. 重新布局，主要是调用 Adapter.notifyDataSetChange 且 Adapter 的 hasStableIds 方法返回为 false 时调用。从这边也可以看出为什么一般情况下 notifyDataSetChange 效率比其它 notifyXXX 方法低（使用二级缓存及优先级更低的缓存 ），同时也知道了，如果我们设置 Adapter.setHasStableIds(ture) 以及其它相关需要的实现，则可以提高效率（使用一级缓存）
2. 在复用时，从一级缓存里面获取到 ViewHolder，但是此时这个 ViewHolder 已经不符合一级缓存的特点了(比如 Position 失效了，跟 ViewType 对不齐)，就会从一级缓存里面移除这个 ViewHolder，添加到这两级缓存里面 
3. 当调用 removeAnimatingView 方法时，如果当前 ViewHolder 被标记为 remove ,会调用 recycleViewHolderInternal 方法来回收对应的 ViewHolder。调用 removeAnimatingView 方法的时机表示当前的 ItemAnimator 已经做完了 

## 总结
到这里，RecyclerView 的缓存复用机制就分析完了，总结一下：
- RecyclerView 的缓存复用机制，主要是通过内部类 Recycler 来实现
- Recycler 有 4 级缓存，每一级的缓存都有各自的作用，会按优先级使用。
- ViewHolder 会从某一级缓存移至其它级别的缓存
- mHideenViews 的存在是为了解决在动画期间进行复用的问题。
- 缓存复用 ViewHolder 时会针对内部不同的状态 ( mFlags ) 进行相应的处理。

本文源码基于：**recyclerview:1.2.0-alpha03**

