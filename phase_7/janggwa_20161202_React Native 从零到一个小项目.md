

# React Native 从零到一个小项目

## 前言

前阵子开始学习 React Native，一个人摸滚带爬的也算是能写个[小项目](https://github.com/JangGwa/ReactNativeMovie)了，在这里分享一下自己从零开始学习的过程，也推荐一些比较优秀的学习资源，让大家学习过程可以提高一些效率。

## 在路上

### 一、环境搭建和 IDE 选型

React Native 环境搭建可以看[官网](http://facebook.github.io/react-native/docs/getting-started.html),也可以去看笔者的[上篇文章](http://www.jianshu.com/p/4d20a1ae7148)，接下去是IDE 选型和配置的环节，有几种比较不错的 IDE：

- **Atom + Nuclide**：Atom 本质上是一个文本编辑器，而不是一个 IDE，因此在用来开发 React Native 时需要配合 Nuclide 一起使用。
- **Sublime Text 3**： 功能十分强大，主要在于它的插件机制。通过 Package Control 功能，可以安装各种需要的插件。
- **WebStorm**：WebStorm 是著名的 JetBrains 公司开发的号称最智能的 Javascript 集成开发环境。
- **Visual Studio Code**：Visual Studio Code 是微软推出的一个轻量级的开源 Web 集成开发环境。

笔者选择了 WebStorm ，只有一个原因，那就是它和 Andoid Studio 很像(笔者是 Andorid Developer)。

### 二、学习基础知识

因为没接触过 Web 前端这一块，所以第一天去学习了 JavaScript、html 和 css 一些基础的知识。记得那天疯狂的找学习的资料，刚开始在极客学院、慕课网这些学习网站上学习了一会，后面在知乎上还是 google 搜索到阮一峰大神的 [es6](http://es6.ruanyifeng.com/) 教程，非常不错。同时，也推荐 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)，也是不错的学习 JavaScript 的地方。当时学习了一两天这些基础的知识，有了初步的了解。

### 三、了解 React Native

接下去就开始疯狂的搜索 React Native 的学习资料，因为完全不知道怎么去写 React Native，所以找到了一些免费的视频学习资源，如果你想要一些学习资源的话，可以去 [React Native 学习指南](https://github.com/JangGwa/react-native-guide)看看，这边概括了很多优秀的学习资源，非常不错。说实话看视频效率实在有点低，你会发现视频只是在逐个讲解讲官网的组件和 Api ，所以笔者看了几天视频，对 React Native 有了初步了解后就没再接着看视频学习了。

### 四、学习优秀项目

当时笔者对 React Native 有了一点了解后，便去 GitHub 上寻找一些优秀的项目对其进行学习。这边给个[网站](https://github.com/crazycodeboy/react-native-awesome)，里面概括了一些优秀的项目。大家可以找一些 star 比较多的，然后更新时间比较近的项目进行学习，笔者是通过[Gank.io](https://github.com/Bob1993/React-Native-Gank)项目进行学习。关于如何学习，给个建议就是 clone 到本地后，按照项目作者的时间线去学习。当然学习过程中会遇到一些你不明白的知识点，那正是你去巩固 JavaScript 或是 React 知识的时候，这样巩固学习比光看书学习效率高多了。

### 五、写一个项目

开始单独写一个项目练手，可以是一个真实项目，也可以是一个小项目用开放的 Api 去完成。笔者写的[项目](https://github.com/JangGwa/ReactNativeMovie)是通过豆瓣 Api 实现的，UI 界面大家可以自行充当设计师去设计，图标可以去 [Iconfont](http://www.iconfont.cn/plus) 找。写完这个练手项目，你会更清楚如何去学习 React Native 。

## 项目介绍

首先看下效果图：

<img src="http://github.com/JangGwa/ReactNativeMovie/blob/master/pic/首页.png" width="320"><img src="https://github.com/JangGwa/ReactNativeMovie/blob/master/pic/推荐.png" width="320"><img src="https://github.com/JangGwa/ReactNativeMovie/blob/master/pic/电影详情.png" width="320"><img src="https://github.com/JangGwa/ReactNativeMovie/blob/master/pic/我的.png" width="320">

项目中运用了一些常用的第三方库，关于第三方库大家可以去 [js.coach](https://js.coach/react-native#content) 查找。接下去对项目的界面和功能进行相应的介绍。

1.界面下方的主 tab

项目中有三个主 tab：首页、推荐和我的，通过 **react-native-scrollable-tab-view** 实现。使用示例如下：

```javascript
render() {
    return (
        <TabNavigator tabBarStyle={{ height: 60, overflow: 'hidden' }}>
          <TabNavigator.Item
              selected={this.state.selectedTab === 'home'}
              title="首页"
              titleStyle={{color: 'rgb(169,183,183)', fontSize: 11}}
              selectedTitleStyle={{color: 'rgb(234,128,16)', fontSize: 11}}
              renderIcon={() => <Image source={require('./images/home_normal.png')}/>}
              renderSelectedIcon={() => <Image source={require('./images/home_selected.png')}/>}
              //badgeText ="1"
              onPress={() => this.setState({selectedTab: 'home'})}
          >
            < Home {...this.props}/>
          </TabNavigator.Item>

          <TabNavigator.Item
              selected={this.state.selectedTab === 'recommend'}
              title="推荐"
              titleStyle={{color: 'rgb(169,183,183)', fontSize: 11}}
              selectedTitleStyle={{color: 'rgb(234,128,16)', fontSize: 11}}
              renderIcon={() => <Image source={require('./images/recommend_normal.png')}/>}
              renderSelectedIcon={() => <Image source={require('./images/recomm_selected.png')}/>}
              onPress={() => this.setState({selectedTab: 'recommend'})}
          >
            < Recommend {...this.props} />
          </TabNavigator.Item>

          <TabNavigator.Item
              selected={this.state.selectedTab === 'mine'}
              title="我的"
              titleStyle={{color: 'rgb(169,183,183)', fontSize: 11}}
              selectedTitleStyle={{color: 'rgb(234,128,16)', fontSize: 11}}
              renderIcon={() => <Image source={require('./images/mine_normal.png')}/>}
              renderSelectedIcon={() => <Image source={require('./images/mine_selected.png')}/>}
              onPress={() => this.setState({selectedTab: 'mine'})}
          >
            < Mine { ...this.props }/>
          </TabNavigator.Item>

        </TabNavigator>

    );
  }
```

2.首页的图片轮播，使用了 **react-native-viewpager** 实现。使用示例如下：

定义数据源

```javascript
this.state = {
      dataSource: new ViewPager.DataSource({
        pageHasChanged: (p1, p2) => p1 !== p2
      })
    };

 this.setState({
        dataSource: this.state.dataSource.cloneWithPages(contentData),
      });
```

应用 ViewPager 组件

```javascript
<ViewPager
	dataSource={this.state.dataSource}
	renderPage={this.renderPage}
	isLoop={true}
	autoPlay={true}>
</ViewPager>
```

显示每个 Page 界面

```javascript
_renderPage(data) {
    return (
        <Image style={ styles.pager } source={{uri: data.cover}}/>
    );
  }
```

3.推荐列表的呈现，ListView 的运用，使用示例如下：

设置数据源

```javascript
this.state = {
      dataSource: new ListView.DataSource({
      rowHasChanged: (r1, r2) => r1 !== r2,
      }),
      data: null,
      loaded: false,
      isRefreshing: false,
      loadMore: false,
      start: 0,
      count: 20,
    };
```

应用 ListView 组件

```javascript
<ListView
	dataSource = {this.state.dataSource}
	renderRow = {this._renderItem.bind(this)}
	onEndReached={this._loadMore.bind(this)}
	renderFooter={this._renderFooter.bind(this)}
	onEndReachedThreshold = {29}
	refreshControl={
		<RefreshControl
		refreshing={this.state.isRefreshing}
		onRefresh={this._refresh.bind(this)}
		tintColor='#aaaaaa'
		title='Loading...'
		progressBackgroundColor='#aaaaaa'/>
	}>
</ListView>
```

4.导航条的设置，使用 **react-native-navbar** 实现。使用示例如下：

```javascript
<NavigationBar
	style = {{height:40}}
	title={{title: '首页'}}
/>
```

5.网络数据的获取，使用示例如下：

```javascript
async fetchData() {
    let response = await fetch(API_TOP);
    let responseJson = await response.json();
    let responseData = responseJson.subjects;
    this.setState({
      data: responseData,
      dataSource: this.state.dataSource.cloneWithRows(responseData),
      loaded: true,
      isRefreshing: false,
    });
  }
```

## 总结

项目中还需要进行完善优化，后续会学习redux对架构进行修整，也会学习性能方面对项目进行修改，也会增添一些动画效果。希望能和大家一起学习，一起进步。

[项目地址](https://github.com/JangGwa/ReactNativeMovie)