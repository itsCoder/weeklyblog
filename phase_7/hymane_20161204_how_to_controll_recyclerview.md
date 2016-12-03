---
title: RecyclerView 入门教程
date: 2016-11-33 21:33:20
categories: Android
tags: [Android, RecyclerView]
---

# 引言
想必大家都用过 ListView，用过它的朋友应该也知道关于 ListView 的 item 复用，在使用 ListView 的时候必须要要考虑使用 ViewHolder 来优化 ListView 性能。好的东西就应该成为“标准”，谷歌工程师将这种 ViewHolder 复用机制纳入官方 SDK，并将其设为标准，随后便产生了 RecyclerView。那么，在能够熟练使用 ListView 的情况下，我还需要去学习、使用 RecyclerView 吗？答案是肯定的，因为我们要向前看。
> 高手在民间

# 简介
RecyclerView 是 V7 包下新增的控件，它是 ListView、GridView 控件的替代品，它标准化了 ViewHolder 类，它大大提高了视图列表展示性能，而且使用起来更加灵活，高度的可定制性。RecyclerView 不能单独来使用，需要和 LayoutManager，ItemDecoration，ItemAnimator，RecyclerView.Adapter 等类配合着使用，即可实现所有你想要的样式。

- 如果你想控制列表的样式，列表点击事件等，请在 RecyclerView.Adapter 里面复写对应方法，后面将详细介绍。
- 如果你想控制列表的显示方式或者布局样式，请使用 LayoutManager。
- 如果你想控制 Item 之间的间隔、分割线，请使用 ItemDecoration。
- 如果你想控制 Item 的增、删、移动的动画，请使用 ItemAnimator。
- 如果你想实现 类似 ViewPager 的效果，Item 居中、居左、居右效果，请使用 SnapHelper。[详情](https://github.com/rubensousa/RecyclerViewSnap)
- 如果你想实现 Item 的滑动和拖拽，请使用 ItemTouchHelper，详情请移步[RecyclerView完美实现拖拽，滑动删除，撤销删除](http://hymane.itscoder.com/2016/08/21/RecyclerView%E5%AE%8C%E7%BE%8E%E5%AE%9E%E7%8E%B0%E6%8B%96%E6%8B%BD%E3%80%81%E6%BB%91%E5%8A%A8%E5%88%A0%E9%99%A4%E4%BB%A5%E5%8F%8A%E6%92%A4%E9%94%80%E5%88%A0%E9%99%A4/)

由上可见几乎所有你想实现的功能，都可以通过 RecyclerView 来实现，只不过借助一下其他类来帮个忙而已，区别于 ListView 一个人干了所有的事，最终就造成了代码臃肿、不好维护、不好扩展等问题。用 RecyclerView 来实现大多情况下会比使用 ListView 要简单些，当然，部分功能的 RecyclerView 实现会有点难度。接下来详细介绍 RecyclerView 的使用。

> RecyclerView, A flexible view for providing a limited window into a large data set.  --Android

# 使用
1. 初始化并设置设置项
```java
    //定义全局变量
    @BindView(R.id.recyclerView)
    RecyclerView mRecyclerView;
    @BindView(R.id.swipe_refresh_widget)
    SwipeRefreshLayout mSwipeRefreshLayout;
    private GridLayoutManager mLayoutManager;
    private BookListAdapter mListAdapter;
    private List<BookInfoResponse> bookInfoResponses;

    //初始化变量并设置
    //官方自带刷新控件
    mSwipeRefreshLayout.setColorSchemeResources(R.color.recycler_color1, R.color.recycler_color2,
            R.color.recycler_color3, R.color.recycler_color4);

    //设置布局管理器，spanCount:列数
    mLayoutManager = new GridLayoutManager(getActivity(), spanCount);
    //控制 item 列数占比，类似 TableLayout 的列收缩和扩展，自行取舍
    mLayoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
        @Override
        public int getSpanSize(int position) {
            return mListAdapter.getItemColumnSpan(position);
        }
    });
    //设置布局方向，横向或者竖向，默认 VERTICAL
    mLayoutManager.setOrientation(LinearLayoutManager.VERTICAL);
    //给 RecyclerView 设置布局
    mRecyclerView.setLayoutManager(mLayoutManager);

    //设置 adapter
    mListAdapter = new BookListAdapter(getActivity(), bookInfoResponses, spanCount);
    mRecyclerView.setAdapter(mListAdapter);

    //设置 Item 增加、移除动画，默认动画
    mRecyclerView.setItemAnimator(new DefaultItemAnimator());
    //监听列表滑动，实现上拉加载，自行取舍
    mRecyclerView.addOnScrollListener(new RecyclerViewScrollDetector());
    //监听下拉刷新
    mSwipeRefreshLayout.setOnRefreshListener(this);
```
2. 编写 Adapter 代码
```java
    //创建 item 视图
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view;
        //默认 item 的 view，返回对应的 holder
        if (viewType == TYPE_DEFAULT) {
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_book_list, parent, false);
            return new BookListHolder(view);
        } else {
        //当列表为空的时候显示的 empty 视图，返回对应的 holder
            view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_empty, parent, false);
            return new EmptyHolder(view);
        }
    }

    //控制本列表显示的 item 类型，如 default_item 和 empty_item 两种
    @Override
    public int getItemViewType(int position) {
        if (bookInfoResponses == null || bookInfoResponses.isEmpty()) {
            return TYPE_EMPTY;
        } else {
            return TYPE_DEFAULT;
        }
    }

    //控制 item 的列缩放和伸展
    //比如正常列表一行显示2个 item，当没数据的时候，
    //emptyView 就会只占一列宽度，需要让其占满一行
    public int getItemColumnSpan(int position) {
        switch (getItemViewType(position)) {
            case TYPE_DEFAULT:
                return 1;
            default:
                return columns;
        }
    }

    //给 item 设置数据，已经监听点击事件
     @Override
    public void onBindViewHolder(final RecyclerView.ViewHolder holder, int position) {
        if (holder instanceof BookListHolder) {
            //Tips 2
            final BookInfoResponse bookInfo = bookInfoResponses.get(holder.getAdapterPosition());
            Glide.with(mContext)
                    .load(bookInfo.getImages().getLarge())
                    .into(((BookListHolder) holder).iv_book_img);
            ((BookListHolder) holder).tv_book_title.setText(bookInfo.getTitle());
            ((BookListHolder) holder).ratingBar_hots.setRating(Float.valueOf(bookInfo.getRating().getAverage()) / 2);
            ((BookListHolder) holder).tv_hots_num.setText(bookInfo.getRating().getAverage());
            ((BookListHolder) holder).tv_book_info.setText(bookInfo.getInfoString());
            ((BookListHolder) holder).tv_book_description.setText("\u3000" + bookInfo.getSummary());
            ((BookListHolder) holder).itemView.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    //共享原始动画跳转方式，自行取舍
                    Bundle b = new Bundle();
                    b.putSerializable(BookInfoResponse.serialVersionName, bookInfo);
                    Bitmap bitmap;
                    GlideBitmapDrawable imageDrawable = (GlideBitmapDrawable) ((BookListHolder) holder).iv_book_img.getDrawable();
                    if (imageDrawable != null) {
                        bitmap = imageDrawable.getBitmap();
                        b.putParcelable("book_img", bitmap);
                    }
                    Intent intent = new Intent(UIUtils.getContext(), BookDetailActivity.class);
                    intent.putExtras(b);
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                        if (BaseActivity.activity == null) {
                            UIUtils.startActivity(intent);
                            return;
                        }
                        ActivityOptionsCompat options = ActivityOptionsCompat.
                                makeSceneTransitionAnimation(BaseActivity.activity, ((BookListHolder) holder).iv_book_img, "book_img");
                        BaseActivity.activity.startActivity(intent, options.toBundle());
                    } else {
                        UIUtils.startActivity(intent);
                    }
                }
            });
        }
    }

    @Override
    public int getItemCount() {
        if (bookInfoResponses.isEmpty()) {
            return 1;
        }
        return bookInfoResponses.size();
    }

     class BookListHolder extends RecyclerView.ViewHolder {

        private final ImageView iv_book_img;
        private final TextView tv_book_title;
        private final AppCompatRatingBar ratingBar_hots;
        private final TextView tv_hots_num;
        private final TextView tv_book_info;
        private final TextView tv_book_description;

        public BookListHolder(View itemView) {
            super(itemView);
            iv_book_img = (ImageView) itemView.findViewById(R.id.iv_book_img);
            tv_book_title = (TextView) itemView.findViewById(R.id.tv_book_title);
            ratingBar_hots = (AppCompatRatingBar) itemView.findViewById(R.id.ratingBar_hots);
            tv_hots_num = (TextView) itemView.findViewById(R.id.tv_hots_num);
            tv_book_info = (TextView) itemView.findViewById(R.id.tv_book_info);
            tv_book_description = (TextView) itemView.findViewById(R.id.tv_book_description);
        }
    }

    class EmptyHolder extends RecyclerView.ViewHolder {
        public EmptyHolder(View itemView) {
            super(itemView);
        }
    }
```
3. 请求数据
```java
    //调用网络请求
    bookListPresenter.loadBooks(null, tag, 0, count, fields);
```
4. 数据请求成功，刷新视图
```java
    bookInfoResponses.clear();
    bookInfoResponses.addAll(((BookListResponse) result).getBooks());
    mListAdapter.notifyDataSetChanged();
    //mListAdapter.notifyItemRangeInserted(0,bookInfoResponses.size());
    page++;
```
5. enjoy it

# Tips
RecyclerView 固然好用，但是在使用过程中还是会出现一些问题，接下来，我介绍下我在日常使用 RecyclerView 过程中遇到的一些问题，已经自己的一些解决方法和建议。

1. 一定要给 RecyclerView 设置一个 LayoutManager。
2. 复写 Adapter 里面的 onBindViewHolder(VH holder, int position)方法时，请使用 holder.getAdapterPosition()作为 item 点击事件的位置，而不是直接使用 position 参数。
3. 在给 RecyclerView 添加滑动事件时`addOnScrollListener` ，当你不再使用它的时候记得移除它。`clearOnScrollListeners()`，移除所有滑动事件。
4. 使用 `notifyDataSetChanged()` 更新视图是没有 item 的动画效果的，即使你 `mRecyclerView.setItemAnimator(new DefaultItemAnimator());`，它会刷新所有的 item，请使用 `notifyItemChanged(int)`,`notifyItemInserted(int)`,`notifyItemRemoved(int)`,`notifyItemRangeChanged(int, int)`,`notifyItemRangeInserted(int, int)`,`notifyItemRangeRemoved(int, int)`代替。官方也如此建议我们。
5. item 不能 match_parent？ Adapter 里面创建 View 时候尝试这样写；
    ```
    LayoutInflater.from(parent.getContext()).inflate(R.layout.item_layout, parent, false);
    ```
6. 如上写法，我的 item 还是不能 match_parent，挤在最左侧。请检查是否给 RecyclerView `addItemDecoration()`了，如果是，请在自定义 Decoration 中控制 item 间距，然后创建 holder 时候使用：
    ```
    LayoutInflater.from(parent.getContext()).inflate(R.layout.item_layout, null, false);
    ```

# 总结
RecyclerView 相比 ListView 有其突出的优势，既然官方推出了新的列表展示控件，个人认为我们就应该去使用它，而不能由于自己熟练使用 ListView 以及加上 RecyclerView 使用上的不便捷（其实不然）而拒绝他。
这是一篇 RecyclerView 的入门教程，主要是总结一下 RecyclerView 的使用和一些 Tips，下期将会继续对 RecyclerView 进行详细介绍，以及分享使用过程中遇到到的问题和解决方案。