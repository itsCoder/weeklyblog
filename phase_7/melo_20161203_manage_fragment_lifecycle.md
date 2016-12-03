>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melo](https://itsmelo.github.io/)
>- 审阅者：[]()


死磕 Fragment 的生命周期
==

本文例子中 github 地址：
[github BuzzerBeater 项目链接](https://github.com/itsMelo/BuzzerBeater)

曾经在北京拥挤的13号线地铁上，一名背着双肩包穿着格子衫带着鸭舌帽脚踏帆布鞋的程序员讲了一句：
“我觉得 Fragment 真的太难用了”。从而引起一阵躁动激烈的讨论。

**正方观点：**

Fragment 真的太好用了。要知道因为 Activity 的启动，涉及到 Android 系统对 ActivityManager 的调度，会关联许多资源和进行诸多复杂运算，在一些高端手机上，这个调度的时长甚至会超过 100 ms。反观 Fragment，启动如巧克力入口般顺滑，轻量不消耗手机资源。还可以做成一个个模块，方便 Activity 复用。并且如果要涉及平板的适配，Fragment 更是必不可少。

**反方观点：**

Fragment 难用，属于坑多难填。Fragment 本质上是一个有生命周期的 View，生命周期繁多并且异常难管理，多个 Fragment 嵌套更是坑中之坑（我也遇到过...），连 square 和 FaceBook 都摒弃了 Fragment，更何况我们呢！

好吧，不要吵了，用或者不用，遇到问题如何解决，相信大家心里都有一个自己的答案。结合我自己开发时候遇到的问题，下面我们来总结一下 Fragment 的生命周期管理方式，以及一些技巧和建议。

hide & show
--
先结合一张项目截图，来直观地看看目前我是如何管理 Fragment 的。

![项目截图](http://upload-images.jianshu.io/upload_images/1915184-beb0ccb60fbfee45?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为我们的项目架构是一 Activity 多 Fragment，并且把所有的 Fragment 都装载到了一个 ViewPager 里面，启动一个新的 Fragment 的过程也是 ViewPager 滑动翻页的过程，未来会考虑把这种管理方式总结成文，整理给大家。总之你目前看到的上图界面，都是 Fragment 呈现的，并且点击按钮什么的，也会跳转到下一个 Fragment，这就涉及到了 Fragment 嵌套。

我也写了一个 Demo，去模拟了这个页面的搭建。

这里想多说几句

通过点击下方 Tab  管理页面的方式目前非常主流，这种布局方式事实上是从 iOS 上面借鉴过来的。（反正现在两大系统都是相互学习）在前一阵 google 的 support 25 包也终于推出了官方的 BottomNavigationView ，不过我的 Android Studio 安装 support 25 总是失败，所以在项目中我选用了一个还不错的开源库来做下方的底部导航。

BottomNavigationView 本身有一套 Material Design 的设计规范如下：

[ Material Design Navigation Bottom](https://material.google.com/components/bottom-navigation.html#bottom-navigation-usage)

感兴趣的去阅读一下，以后对产品、设计开撕是很有帮助的。其中有这么一条很有意思，是说 BottomNavigationView 并不建议把它设计成横向滑动的形式，也就是用 ViewPager 去装载 Fragment。为什么说这句很有意思呢？事实上市面上很多主流的 app，它们的 BottomNavigationView 确实是不可以横向滑动的，而我们每个人都在用的，月活8亿的国民软件微信，就恰恰把它的主页面做成了可以横向滑动的。

这里我想说下我的个人看法，首先规范未必需要严格遵守，做什么样的功能实现什么样的效果，要结合自己项目的架构和产品做一个合理的需求。拿 360手机助手 这个 app 举例，它底部的每一个 tab 都搭建了一个非常“重量级”的模块，并且每个 tab 下界面的内部还有许多负责的 View 层级和嵌套滑动的 ViewPager，所以试想一下，这样的页面要是做成微信那个样子，不卡顿就怪了~反观微信，首先我认为它的每个界面层级和交互都不复杂，逻辑也都在页面内，所以做成横向滑动的反而能提升用户的体验。

好了，前面说了好多无关紧要的话，赶紧来看看 demo 中通过 hide 和 show 的方式如何来管理 Fragment。
**MainActivity**

```
    private SearchFragment searchFragment;
    private MusicFragment musicFragment;
    private CarFragment carFragment;
    private SettingFragment settingFragment;

    private BaseFragment currentFragment;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        setSupportActionBar(toolbar);
        initView();
        initData();
        initListener();
    }

    private void initData() {
        if (searchFragment == null) {
            searchFragment = SearchFragment.newInstance(getString(R.string.tab_1));
        }

        if (!searchFragment.isAdded()) {
            // 提交事务
            getSupportFragmentManager().beginTransaction().add(R.id.fl_content, searchFragment).commit();

            // 记录当前Fragment
            currentFragment = searchFragment;
        }
    }

    private void initListener() {
        bottomNavigation.setOnTabSelectedListener(new AHBottomNavigation.OnTabSelectedListener() {
            @Override
            public boolean onTabSelected(int position, boolean wasSelected) {

                Log.d(TAG, "onTabSelected: position:" + position + ",wasSelected:" + wasSelected);

                if (position == 0) {// 导航
                    clickSearchLayout();
                }
                if (position == 1) {// 音乐
                    clickMusicLayout();
                }
                if (position == 2) {// OBD
                    clickCarLayout();
                }
                if (position == 3) {
                    clickSettingLayout();
                }
                return true;
            }
        });
    }
    private void clickSearchLayout() {
        if (searchFragment == null) {
            searchFragment = SearchFragment.newInstance(getString(R.string.tab_1));
        }
        addOrShowFragment(getSupportFragmentManager().beginTransaction(), searchFragment);
    }

    private void clickMusicLayout() {
        if (musicFragment == null) {
            musicFragment = MusicFragment.newInstance(getString(R.string.tab_2));
        }
        addOrShowFragment(getSupportFragmentManager().beginTransaction(), musicFragment);
    }

    private void clickCarLayout() {
        if (carFragment == null) {
            carFragment = CarFragment.newInstance(getString(R.string.tab_3));
        }
        addOrShowFragment(getSupportFragmentManager().beginTransaction(), carFragment);
    }

    private void clickSettingLayout() {
        if (settingFragment == null) {
            settingFragment = SettingFragment.newInstance(getString(R.string.tab_4));
        }
        addOrShowFragment(getSupportFragmentManager().beginTransaction(), settingFragment);
    }


    /**
     * 添加或者显示 fragment
     *
     * @param transaction
     * @param fragment
     */
    private void addOrShowFragment(FragmentTransaction transaction, Fragment fragment) {
        if (currentFragment == fragment)
            return;

        if (!fragment.isAdded()) { // 如果当前fragment未被添加，则添加到Fragment管理器中
            transaction.hide(currentFragment).add(R.id.fl_content, fragment).commit();
        } else {
            transaction.hide(currentFragment).show(fragment).commit();
        }
        currentFragment = (BaseFragment) fragment;
    }
```

代码理解起来非常容易，我通过 add & show / hide 的方式来管理底部的四个 tab 相互切换，并且打印了这四个 Fragment 的所有生命周期方法，包括`onHiddenChanged` 和 `setUserVisibleHint`


当第一次进入某个页面时：

![首次进入某个页面时](http://upload-images.jianshu.io/upload_images/1915184-73843156f8e179ab?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，当我依次点击下方四个 tab，界面第一次加载的时，会走 Fragment 的创建周期 onAttach -- onResume，也许你会问，上面我执行相互切换操作，从第一个页面切换到第二个的时候，为什么第一个 Fragment 页面不可见了，不会调用 onPause 和 onStop 呢？

这是在你了解过 Activity 生命周期并且刚接触 Fragment 的生命周期时，第一个容易陷入的误区，事实上 Fragment 的生命周期，除了第一次它创建或销毁之外，统统都是由 Activity 所驱动的。 举个例子，当我点击 home 键回到桌面时：

![点击 home 键](http://upload-images.jianshu.io/upload_images/1915184-a2f60abb148e85ed?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到我已经加载了的几个 Fragment 齐刷刷的调用了 onPause 和 onStop，如此得步调一致是因为这些 Fragment attach 的 Activity 回调了 onPause 和 onStop。相信肯定会有人说“不对啊，我要是用 replace 的方式去切换 Fragment，我打包票 Fragment 会像 Activity 一样，完整得走完生命周期“

你说的没错，因为 replace 这种切换方式就是始终上面我总结的那句“首次创建或销毁“，并且在 BottomNavigation 这样的使用场景中，没人会用这种 replace 的方式，因为每次切换都要重新创建 Fragment，用户看了下流量估计会打 12315 了。

（如果 Fragment 代表前男/女友，据说男人是用 add 保存，女人使用 replace 替换 hhh...）

当底部的四个 Fragment 都已经加载完成之后咧？再一起看下 log：

![onHiddenChanged 回调](http://upload-images.jianshu.io/upload_images/1915184-e07a6adc37fe2be8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当底部四个 Fragment 全部创建入栈之后，show 和 hide 来管理 Fragment，此时只有 `onHiddenChanged` 方法回调，这时候显然你可以在 `hide = false` 时，做一些资源回收操作，在 `hide = true` 时，做一些刷新操作。

在刚才我们打印的方法中，好像有一个一直没出现，没错就是 `setUserVisibleHint`，如果你做过 Fragment+ViewPager 的懒加载（下文我们会讲这个），相信你对它就比较熟悉了，通过名字就能猜测出来，这个方法是我们主动设置给 Fragment 的，那我们就来试试看：

改造 `addOrShowFragment` 

```
    /**
     * 添加或者显示 fragment
     *
     * @param transaction
     * @param fragment
     */
    private void addOrShowFragment(FragmentTransaction transaction, Fragment fragment) {
        if (currentFragment == fragment)
            return;

        if (!fragment.isAdded()) { // 如果当前fragment未被添加，则添加到Fragment管理器中
            transaction.hide(currentFragment).add(R.id.fl_content, fragment).commit();
        } else {
            transaction.hide(currentFragment).show(fragment).commit();
        }

        currentFragment.setUserVisibleHint(false);

        currentFragment = (BaseFragment) fragment;

        currentFragment.setUserVisibleHint(true);

    }
```
在切换时，我们对上一个 fragment `setUserVisibleHint`设置为 false，要展现的 Fragment `setUserVisibleHint` 设置为 true，打印 log 看看：

![setUserVisibleHint](http://upload-images.jianshu.io/upload_images/1915184-4449ec0891b3f069?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到目前我们用 setUserVisibleHint 也达到了与 onHiddenChanged 一样的效果。

文章写到现在，我们来做一个总结，上文的这种方式就是主流通过 BottomNavigation 管理 Fragment 的方式，这种方式有什么好处呢？毫无疑问会节省资源，不点开的界面不去创建它（这一点 ViewPager 做不到），只创建一次，未来仅仅更新界面数据就可以了。

ViewPaager & Fragment
--

ViewPager 和 Fragment 配合使用相信大多数人都很熟悉了，所以来快速地给大家总结一下我认为需要梳理清楚的几个知识点，先来看我搭建的页面：

![TabLayout+ViewPager](http://upload-images.jianshu.io/upload_images/1915184-9d4ecb7454abbf8f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我在导航的这个模块中，搭建了一个 TabLayout+ViewPager+Fragment 的页面结构，当启动 app，进入首页，各个 Fragment 的生命周期方法是怎样的呢？

![首页和 Tab 页生命周期方法](http://upload-images.jianshu.io/upload_images/1915184-06aec47f6157e0aa?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，当我进入 app 的时候，所有 Tab 的父容器 SearchFragment 创建，调用了 onAttach --> onResume 这当然是我们预料之中的，我们 ViewPager 第一个装载并展示的是 ScienceFragment，ScienceFragment 创建没有问题，可是第二个 tab GameFragment 为什么加载了呢？

没错这就是 ViewPager 的预加载机制。

ViewPager 出于优化体验的好心，默认去加载相邻两页，来尽可能保证滑动的流畅性，可是假如我们这是一个新闻资讯类的 app，每一个 tab 涉及了复杂的页面和大量的网络请求，这种预加载的机制带来的可能就是麻烦了。所以我们寻找一些办法试图去掉 ViewPager 的预加载。

ViewPager 自身提供了一个方法，`mViewPager.setOffscreenPageLimit()`，这个方法的官方文档的解释：

![setOffscreenPageLimit 官方文档](http://upload-images.jianshu.io/upload_images/1915184-b50d507d34361c7b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

它的意思就是设置 ViewPager 左右两侧缓存页的数量，默认是1，那我们给它设置为0，是不是就能取消预加载了呢？再看看这段蜜汁源码：

![setOffscreenPageLimit 源码](http://upload-images.jianshu.io/upload_images/1915184-41aeb6dc992ac5b9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总之源码的意思就是你设置为小于1的值就默认为1，反正这条路目前行不通了。

当然还有一个办法，你直接修改源码以后重新打一个 v4 包，不过非常不建议这样做，未来会产生一些兼容问题。

好吧，你应该知道马上就要说 ViewPager 的懒加载了， 就是要用到上文我们提到的 setUserVisibleHint 方法，当我左右滑动时，来看看打印的 log：

![滑动时 log](http://upload-images.jianshu.io/upload_images/1915184-a1a256101fcdd443?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从 log 中可以分析到两个问题，首先 setUserVisibleHint 这个方法可能会在 onAttach 之前就调用，其次在滑动中设置缓存页数之外的页确实是销毁了。

回顾下前文， 明明说 setUserVisibleHint 这个方法需要主动调用，那在 ViewPager 中，Fragment 的 setUserVisibleHint 方法是谁在何时调用的呢？

我的 ViewPager Adapter 在这里：

```
public class MainTabAdapter extends FragmentStatePagerAdapter {

    private List<Fragment> mList;

    private String mTabTitle[] = new String[]{"科技", "游戏", "装备", "创业", "想法"};

    public MainTabAdapter(FragmentManager fm, List<Fragment> list) {
        super(fm);
        mList = list;
    }

    @Override
    public Fragment getItem(int position) {
        return mList.get(position);
    }

    @Override
    public int getCount() {
        return mList.size();
    }

    @Override
    public CharSequence getPageTitle(int position) {
        return mTabTitle[position];
    }

}
```
看来看去也就 FragmentStatePagerAdapter  能做这件事了，点进去看看：

FragmentStatePagerAdapter 这个类其实就是一个对 PagerAdapter 的一个封装类，不到200行的代码，果真找到了 setUserVisibleHint 。

有两处位置调用了 setUserVisibleHint ，第一个位置：

![instantiateItem](http://upload-images.jianshu.io/upload_images/1915184-86b3cdad740a50ef?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二个位置：

![setPrimaryItem](http://upload-images.jianshu.io/upload_images/1915184-303808d658d7e612?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里源码处理的逻辑是这样子的：

在 instantiateItem 方法中，在我的数据集合中取出对应 positon 的 Fragment，直接给它的 setUserVisibleHint 设置为 false，然后才把它 add 进 FragmentTransaction 中，这恰恰解释了为什么 setUserVisibleHint 的第一次调用是在 onAttach 之前。

下一步 setUserVisibleHint 的设置是在 `setPrimaryItem`中，`setPrimaryItem`这个方法可以得到当前 ViewPager 正在展示的 Fragment，并且将上一个 Fragment 的 setUserVisibleHint 置为 false，将要展示的 setUserVisibleHint 置为 true。

通过阅读源码，我们明白了原理，所以直接给大家上在 BaseFragment 实现懒加载代码：

```
public class BaseFragment extends Fragment {

    protected boolean isViewCreated = false;

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return super.onCreateView(inflater, container, savedInstanceState);
    }

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        if (getUserVisibleHint()) {
            lazyLoadData();
        }
    }

    /**
     * 懒加载方法
     */
    protected void lazyLoadData() {
        
    }
}
```

在具体的 Fragment 中实现懒加载
```
public class ScienceFragment extends BaseFragment {

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_science, container, false);
        Log.e(TAG, "onCreateView");
        isViewCreated = true;
        return view;
    }

    @Override
    protected void lazyLoadData() {
        super.lazyLoadData();
        if (isViewCreated) {
            Log.e(TAG, "lazyLoadData...");
        }
    }
    
}
```

看下 log：

![懒加载 log](http://upload-images.jianshu.io/upload_images/1915184-e5ca14d9d1143cba?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，懒加载就这样实现了。

这里我们再做一个阶段总结，首先大家心里要清楚 setUserVisibleHint 这个是 ViewPager 的行为，它始终都是先行与 Fragment 的生命周期调用的。我们之所以能用懒加载这种办法，主要是因为预加载的 Fragment 已经创建完成一路调用了 onAttach --> onPause，也就是说这个 Fragment 此时可用的，懒加载才有理由生效。不知道这样描述是否难懂，但是跑一下本文的例子就肯定能明白上面这段话了。

所以当 Fragment 第一次创建时，懒加载不会同时调用，所以我们来继续优化优化，我们让 ViewPager 一起加载这五个 Fragment 的布局，然后懒加载就全程可用啦~

```
  mViewPager.setOffscreenPageLimit(5);
```

此时无论是我滑动还是点击上方 tab 跳转到任意一个页面，`lazyLoadData` 方法都会调用，我们可以先加载布局出来，然后可见时刷新数据就 OK 了。
 
关于 Fragment 的管理，主要就是上文的两种方式，当然具体到每个人自己的项目时，需要分析需求和产品思路找到一个适合自己的方式。当我们知道原理时，做操作就心里有底多了，出了问题也可以快速定位。

上面我们 ViewPager 的 adapter 使用的是 FragmentStatePagerAdapter，还有一个 FragmentPagerAdapter，因为本文的篇幅有些过长，下次会总结出它们在源码角度的区别，以及使用过程中踩到的一些坑。（如果有一些奇怪的问题无法解决，建议先使用 FragmentStatePagerAdapter）。


**写在最后：**

我们回过头来看开始那个辩题， Fragment 到底用不用？对于大多数开发者来说，当然要用，我个人其实还非常喜欢 Fragment，使用 Fragment 能体现 Android 组件化的思想，其带给开发者的便利远大于麻烦。虽然其生命周期复杂，栈又奇怪难管理，不过当正确的姿势使用 Fragment（不要嵌套 Fragment 使用），趟过一个个坑时，对 Fragment 自然也有信心了。

最后再上一张 Fragment 的生命周期图吧~

![fragment 生命周期](http://upload-images.jianshu.io/upload_images/1915184-b0185314cdb61adc?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)