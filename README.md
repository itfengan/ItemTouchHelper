# ItemTouchHelper

#### RecyclerView结合ItemTouchHelper实现的列表和网格布局的拖拽效果

那么是如何实现的呢？主要就要使用到ItemTouchHelper ，ItemTouchHelper 是support-v7包中加入的一个帮助开发人员处理拖拽和滑动的实现类，它能够让你非常容易实现侧滑删除、拖拽的功能。

#### 我们只需要实例化一个ItemTouchHelper，然后关联到RecyclerView就OK了：

```java
itemTouchHelper = new ItemTouchHelper(new ItemTouchHelper.Callback());
itemTouchHelper.attachToRecyclerView(recyclerView);
```

#### 构造方法中需要一个ItemTouchHelper.Callback，ItemTouchHelper会在拖拽或剔除的时候回调Callback中相应的方法，我们只需在Callback中实现自己的逻辑就可以了。

```java
@Override
    public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
    }

    @Override
    public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
    }

    @Override
    public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
    }
```

getMovementFlags用于设置是否处理拖拽事件和滑动事件，以及拖拽和滑动操作的方向，比如如果是列表类型的RecyclerView，拖拽只有UP、DOWN两个方向，而如果是网格类型的则有UP、DOWN、LEFT、RIGHT四个方向：

```java
@Override
    public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
        if (recyclerView.getLayoutManager() instanceof GridLayoutManager) {
            final int dragFlags = ItemTouchHelper.UP | ItemTouchHelper.DOWN | ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT;
            final int swipeFlags = 0;
        } else {
            final int dragFlags = ItemTouchHelper.UP | ItemTouchHelper.DOWN;
            final int swipeFlags = 0; 
        }
        return makeMovementFlags(dragFlags, swipeFlags);
    }
```

dragFlags 是拖拽标志，swipeFlags是滑动标志，我们把swipeFlags 都设置为0，表示不处理滑动操作。

如果我们设置了非0的dragFlags ，那么当我们长按item的时候就会进入拖拽并在拖拽过程中不断回调onMove()方法，我们就在这个方法里获取当前拖拽的item和已经被拖拽到所处位置的item的ViewHolder，有了这2个ViewHolder，我们就可以交换他们的数据集并调用Adapter的notifyItemMoved方法来刷新item

```java
@Override
public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
      int fromPosition = viewHolder.getAdapterPosition();//得到拖动ViewHolder的position
      int toPosition = target.getAdapterPosition();//得到目标ViewHolder的position
      if (fromPosition < toPosition) {
          for (int i = fromPosition; i < toPosition; i++) {
              Collections.swap(results, i, i + 1);
          }
      } else {
          for (int i = fromPosition; i > toPosition; i--) {
              Collections.swap(results, i, i - 1);
          }
      }
      adapter.notifyItemMoved(fromPosition, toPosition);
      return true;
}
```

同理如果我们设置了非0的swipeFlags，我们在滑动item的时候就会回调onSwiped的方法，我们不处理这个事件，空着就行了。

到这里，已经可以拖拽了，但是拖拽的时候我们拖拽的对象不能高亮显示，这是不友好的，我们希望拖拽的Item在拖拽的过程中背景颜色加深，这样就需要继续重写下面两个方法

```java
//当长按选中item的时候（拖拽开始的时候）调用
    @Override
    public void onSelectedChanged(RecyclerView.ViewHolder viewHolder, int actionState) {
    }

    //当手指松开的时候（拖拽完成的时候）调用
    @Override
    public void clearView(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
    }
```

我们在开始拖拽的时候给item添加一个背景色，然后在拖拽完成的时候还原：

```java
@Override
    public void onSelectedChanged(RecyclerView.ViewHolder viewHolder, int actionState) {
        if (actionState != ItemTouchHelper.ACTION_STATE_IDLE) {
            viewHolder.itemView.setBackgroundColor(Color.LTGRAY);
        }
        super.onSelectedChanged(viewHolder, actionState);
    }

    @Override
    public void clearView(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
        super.clearView(recyclerView, viewHolder);
        viewHolder.itemView.setBackgroundColor(0);
    }
```

如何现在某些条目不参与拖拽或者滑动

Callback类中有一个方法我们没有重写：

```java
@Override
    public boolean isLongPressDragEnabled() {
        return true;
    }
```

这个方法是为了告诉ItemTouchHelper是否需要RecyclerView支持长按拖拽，默认返回是ture（即支持），理所当然我们要支持，所以我们没有重写，因为默认true。但是这样做是默认全部的item都可以拖拽，怎么实现部分item拖拽呢，查阅isLongPressDragEnabled方法的源码发现，上面的注释上写着：

> Default value returns true but you may want to disable this if you want to start 
> dragging on a custom view touch using {@link #startDrag(ViewHolder)}.

意思是如果你想自定义触摸view，那么就使用startDrag(ViewHolder)方法。

原来如此，我们可以在item的长按事件中得到当前item的ViewHolder ，然后调用ItemTouchHelper.startDrag(ViewHolder vh)就可以实现拖拽了，那就这么办：

首先我们重写isLongPressDragEnabled返回false，我们要自己调用拖拽过程：

```java
 @Override
    public boolean isLongPressDragEnabled() {
        return false;
    }
```

接着我们给RecyclerView添加item长按事件，判断item是否是最后一个（最后一个是“更多”），不是则开始拖拽。

但是，我们都知道RecyclerView并没有提供OnItemLongClickListener，这个问题我在上一篇博客中已经完美地解决了，就是使用OnItemTouchListener，然后识别触摸手势，这里给上传送门：[RecyclerView无法添加onItemClickListener最佳的高效解决方案](http://blog.csdn.net/liaoinstan/article/details/51200600)，后面我就直接使用上一篇的成果，不重复讲了：

```java
recyclerView.addOnItemTouchListener(new OnRecyclerItemClickListener(recyclerView) {
    //item 长点击事件
    @Override
    public void onLongClick(RecyclerView.ViewHolder vh) {
        //如果item不是最后一个，则执行拖拽
        if (vh.getLayoutPosition()!=results.size()-1) {
            itemTouchHelper.startDrag(vh);
        }
    }
    //item 点击事件
    @Override
    public void onItemClick(RecyclerView.ViewHolder vh) {
    }
});
```

在demo中SimpleItemTouchHelperCallback.java类中有下列方法

```java
public void filterPosition(List<Integer> filterPos) {
    if (filterPos!=null) {
        this.filterPos.clear();
        this.filterPos.addAll(filterPos);
    }
}
```

来过滤掉部分item

## 保存位置

关闭页面以后再打开，又恢复到了初始化的位置，所以就需要保存调整的位置到本地，下次初始化的时候读取位置。 
保存位置应该由开发者自己实现，因为每个人本地化数据的方式都不一样，我这里做一个简单的实现，使用了开源的ACache类，两个方法，搞定：

```java
//读取
ACache.get(context).getAsObject("items");
//存储
ACache.get(context).put("items",results);
```

在clearView方法（拖拽完成）中调用存储方法，在页面初始化数据是调用读取方法。

#### 震动

支付宝的拖拽网格在长按后开始拖拽时会有一次短时间的震动提示用户开始拖拽了，很友好的交互，我们也加一个：

添加权限：

```xml
<uses-permission android:name="android.permission.VIBRATE" />
```

在开始拖拽时添加下面代码：

```java
//获取系统震动服务
Vibrator vib = (Vibrator) activity.getSystemService(Service.VIBRATOR_SERVICE);
//震动70毫秒
vib.vibrate(70);
```
