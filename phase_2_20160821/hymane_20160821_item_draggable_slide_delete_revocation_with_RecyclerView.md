---
title: RecyclerView完美实现拖拽，滑动删除，撤销删除
date: 2016-08-21 19:50:19
categories: Android
tags: [Android ,RecyclerView ,swipe]
---

> 本文所讲的都是使用自带 **API** 实现，并不借用第三方控件。用于**RecyclerView** 中实现*滑动删除*，*拖拽排序*，以及如何实现删除后*撤销操作*（类似于知乎中撤销删除操作）
 - 初始化RecyclerView，绑定Adapter，LayoutManager等。

#效果图

![效果图](http://img.blog.csdn.net/20160319150453310)

#直接上代码

```
	//数据
	mUserBookShelfResponses = new ArrayList<>();
	mRecyclerView =(RecyclerView)findViewById(R.id.recyclerView);
	mSwipeRefreshLayout = (SwipeRefreshLayout)findViewById(R.id.swipe_refresh_widget);
	//给刷新控件添加颜色，最多4中颜色
	mSwipeRefreshLayout.setColorSchemeResources(R.color.recycler_color1,R.color.recycler_color2,R.color.recycler_color3, R.color.recycler_color4);
	//瀑布流
	mLayoutManager = new StaggeredGridLayoutManager(2,StaggeredGridLayoutManager.VERTICAL);
	mRecyclerView.setHasFixedSize(false);
	mRecyclerView.setLayoutManager(mLayoutManager);
	//创建Adapter
	mBookShelfAdapter = new UserBookShelfAdapter(mUserBookShelfResponses);
	//设置Item增加、移除动画
	mRecyclerView.setItemAnimator(new DefaultItemAnimator());
	final int space = DensityUtils.dp2px(getActivity(), 4);
	//自定义一个Decoration,用于recyclerView中每一个Item之间的间隔（后面将贴出代码【附1】）
	mRecyclerView.addItemDecoration(new StaggeredGridDecoration(space, space, space, space));
	mRecyclerView.setAdapter(mBookShelfAdapter);
	//绑定监听事件实现上拉加载更多【附2】
	mRecyclerView.addOnScrollListener(new RecyclerViewScrollDetector());
	//监听刷新
	mSwipeRefreshLayout.setOnRefreshListener(this);
```

- 使用 ItemTouchHelper 工具类来处理RecyclerView中item的选中、滑动或（和）拖拽动作。
 Google官方文档上是这么介绍的：
This is a utility class to add swipe to dismiss and drag & drop support to RecyclerView.
意思就是：这是一个支持RecyclerView滑动删除和拖拽的实用工具类
 

####看看它的构造函数：

```
 public ItemTouchHelper(Callback callback) {
        mCallback = callback;
    }
```
我们发现这里需要一个Callback参数，进入ItemTouchHelper我们会看到有一个Callback类，集成它需要实现一些方法

```
		//所有动作的标志
		@Override
        public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
            return 0;
        }

		//拖拽
        @Override
        public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
            return false;
        }

		//滑动
        @Override
        public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {

        }
```
我们还会发现还有一个已经被封装好了的一个SimpleCallback类，Google工程师已经为我们封装好了一些操作，让我们更简单的使用它。

```
ItemTouchHelper.Callback mCallback = new ItemTouchHelper.SimpleCallback(int dragDirs, int swipeDirs) {
    /**
     * @param recyclerView
     * @param viewHolder 拖动的ViewHolder
     * @param target 目标位置的ViewHolder
     * @return
     */
    @Override
    public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
        //...
        return false;
    }
    /**
     * @param viewHolder 滑动的ViewHolder
     * @param direction 滑动的方向
     */
    @Override
    public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
        //...
    }
};
```
这里来解释一下构造函数SimpleCallback(int dragDirs, int swipeDirs)中的参数的含义，有一下这些可选项：

```
	/**
     * Up direction, used for swipe & drag control.
     */
    public static final int UP = 1;

    /**
     * Down direction, used for swipe & drag control.
     */
    public static final int DOWN = 1 << 1;

    /**
     * Left direction, used for swipe & drag control.
     */
    public static final int LEFT = 1 << 2;

    /**
     * Right direction, used for swipe & drag control.
     */
    public static final int RIGHT = 1 << 3;
```
即我们对哪些方向操作关心。如果我们关心用户向上拖动，可以将dragDirs参数填充UP | DOWN ，如果我们对左右滑动感兴趣，填充swipeDirs参数为
LEFT | RIGHT 。0表示从不关心。

#### 然后调用attachToRecyclerView()绑定动作

```
ItemTouchHelper touchHelper = new ItemTouchHelper(newSimpleItemTouchHelperCallback(0,ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT));
//attachToRecyclerView(将动作处理监听绑定到RecyclerView中)
touchHelper.attachToRecyclerView(mRecyclerView);
```

#### 接下来我们看看Callback具体实现

```
class SimpleItemTouchHelperCallback extends ItemTouchHelper.SimpleCallback {
        //保存被删除item信息，用于撤销操作
        //这里使用队列数据结构，当连续滑动删除几个item时可能会保存多个item数据，并需要记录删除循序。
        BlockingQueue queue = new ArrayBlockingQueue(3);

        public SimpleItemTouchHelperCallback(int dragDirs, int swipeDirs) {
            super(dragDirs, swipeDirs);
        }

        @Override
        public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
            //得到拖动ViewHolder的position
            int fromPosition = viewHolder.getAdapterPosition();
            //得到目标ViewHolder的position
            int toPosition = target.getAdapterPosition();
            if (fromPosition < toPosition) {
                //分别把中间所有的item的位置重新交换
                for (int i = fromPosition; i < toPosition; i++) {
                    Collections.swap(mUserBookShelfResponses, i, i + 1);
                }
            } else {
                for (int i = fromPosition; i > toPosition; i--) {
                    Collections.swap(mUserBookShelfResponses, i, i - 1);
                }
            }
            mBookShelfAdapter.notifyItemMoved(fromPosition, toPosition);
            //返回true表示执行拖动
            return true;
        }

        @Override
        public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
	        //记录将要删除item的位置
            int position = viewHolder.getAdapterPosition();
            final UserBookShelfResponse bookShelfResponse = mUserBookShelfResponses.get(position);
            bookShelfResponse.setIndex(position);
            //将位置放到数据中，再保存到队列中方便操作
            queue.add(bookShelfResponse);
            //滑动删除，将该item数据从集合中移除，
            //被移除的数据可能还需要被撤销，已经被保存到队列中了
            mUserBookShelfResponses.remove(position);
            mBookShelfAdapter.notifyItemRemoved(position);
        }
		//处理动画
        @Override
        public void onChildDraw(Canvas c, RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, float dX, float dY, int actionState, boolean isCurrentlyActive) {
            if (actionState == ItemTouchHelper.ACTION_STATE_SWIPE) {
                //滑动时改变Item的透明度，以实现滑动过程中实现渐变效果
                final float alpha = 1 - Math.abs(dX) / (float) viewHolder.itemView.getWidth();
                viewHolder.itemView.setAlpha(alpha);
                viewHolder.itemView.setTranslationX(dX);
            } else {
                super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive);
            }
        }

		//滑动事件完成时回调
		//在这里可以实现撤销操作
        @Override
        public void clearView(final RecyclerView recyclerView, final RecyclerView.ViewHolder viewHolder) {
            super.clearView(recyclerView, viewHolder);
            if (!queue.isEmpty()) {
		        //如果队列中有数据，说明刚才有删掉一些item
                Snackbar.make(((BaseActivity) getActivity()).getToolbar(), R.string.delete_bookshelf_success, Snackbar.LENGTH_LONG).setAction(R.string.repeal, new View.OnClickListener() {
                    @Override
                    public void onClick(View v) {
                    //SnackBar的撤销按钮被点击，队列中取出刚被删掉的数据，然后再添加到数据集合，实现数据被撤回的动作
                        final UserBookShelfResponse bookShelfResponse = (UserBookShelfResponse) queue.remove();
					//通知Adapter                        mBookShelfAdapter.notifyItemInserted(bookShelfResponse.getIndex());
                        mUserBookShelfResponses.add(bookShelfResponse.getIndex(), bookShelfResponse);
                        //实际开发中遇到一个bug：删除第一个item再撤销出现的试图延迟
                        //手动将recyclerView滑到顶部可以解决这个bug
                        if (bookShelfResponse.getIndex() == 0) {
                            mRecyclerView.smoothScrollToPosition(0);
                        }
                    }
                }).setCallback(new Snackbar.Callback() {
                //不撤销将做正在的删除操作，监听SnackBar消失事件，
                //SnackBar消失（非排挤式消失）出队、访问服务器删除数据。
                    @Override
                    public void onDismissed(Snackbar snackbar, int event) {
                        super.onDismissed(snackbar, event);
                        //event 为消失原因，详细介绍在下文【附3】
                        //排除一种情况就是联系删除多个item SnackBar挤掉前一个SnackBar导致的
                        //消失
                        if (event != DISMISS_EVENT_ACTION) {
                            final UserBookShelfResponse bookShelfResponse = (UserBookShelfResponse) queue.remove();
                            mLibraryPresenter.setBookShelf(BaseApplication.getUserId(), BaseApplication.getUserPassword(),
                                    bookShelfResponse.getName(), bookShelfResponse.getRemark(), bookShelfResponse.getID(), 1);
                        }
                    }
                }).show();
            }
        }

		//是否长按进行拖拽
        @Override
        public boolean isLongPressDragEnabled() {
            return true;
        }
    }
```

> 附1

```
/**
 * Author   :hymanme
 * Email    :hymanme@163.com
 * Create at 2016/1/12
 * Description:
 */
public class StaggeredGridDecoration extends RecyclerView.ItemDecoration {
    private int left;
    private int top;
    private int right;
    private int bottom;

    public StaggeredGridDecoration(int left, int top, int right, int bottom) {
        this.left = left;
        this.top = top;
        this.right = right;
        this.bottom = bottom;
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        outRect.bottom = bottom;
        if (parent.getChildAdapterPosition(view) < 2) {
            outRect.top = 2 * top;
        } else {
            outRect.top = top;
        }
        if (parent.getChildAdapterPosition(view) % 2 == 0) {
            outRect.left = 2 * left;
            outRect.right = right;
        } else {
            outRect.left = left;
            outRect.right = 2 * right;
        }
    }
}
```

> 附2

```
	class RecyclerViewScrollDetector extends RecyclerView.OnScrollListener {
		//如果你的recyclerview是列表只需要用一个int就可以，private int lastVisibleItem;
		//如果是gridview几列就需要几个数据的一个数组,
		//如3列private int[] lastVisibleItem = {0, 0, 0};
        private int[] lastVisibleItem = {0, 0};
        private int mScrollThreshold = DensityUtils.dp2px(x.app(), 1);

        @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
            if (newState == RecyclerView.SCROLL_STATE_IDLE &&
                    (lastVisibleItem[0] + mLayoutManager.getSpanCount() >= mBookShelfAdapter.getItemCount())) {
                onLoadMore();
            }
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            boolean isSignificantDelta = Math.abs(dy) > mScrollThreshold;

            if (isSignificantDelta) {
                if (dy > 0) {
                    ((MainActivity) getActivity()).hideFloatingBar();
                } else {
                    ((MainActivity) getActivity()).showFloatingBar();
                }
            }
            lastVisibleItem = mLayoutManager.findLastCompletelyVisibleItemPositions(null);
        }
    }
```

> 附3

```
		/** Indicates that the Snackbar was dismissed via a swipe.*/
        public static final int DISMISS_EVENT_SWIPE = 0;
        /** Indicates that the Snackbar was dismissed via an action click.*/
        public static final int DISMISS_EVENT_ACTION = 1;
        /** Indicates that the Snackbar was dismissed via a timeout.*/
        public static final int DISMISS_EVENT_TIMEOUT = 2;
        /** Indicates that the Snackbar was dismissed via a call to {@link #dismiss()}.*/
        public static final int DISMISS_EVENT_MANUAL = 3;
        /** Indicates that the Snackbar was dismissed from a new Snackbar being shown.*/
        public static final int DISMISS_EVENT_CONSECUTIVE = 4;
```

> 附4

```
//你还可以复写以下方法定义选中item时动画逻辑
@Overridepublic void onSelectedChanged(RecyclerView.ViewHolder viewHolder, int actionState) {
  super.onSelectedChanged(viewHolder, actionState);
  //当选中Item时候会调用该方法，重写此方法可以实现选中时候的一些动画逻辑
  Log.v("zxy","onSelectedChanged");
}
```

> 总结
RecyclerView 是ListView的很好的替代品，它不仅带给我们更加方便地实现方法，还优化了视图复用效率。原本用listview很难实现的功能用RecyclerView来实现，却变得很简单。吼吼。