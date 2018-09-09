---
title: Appium自动化测试介绍和使用说明
date: 2018-09-09 20:58:43
tags: Appium, Python
category: iOS
---

### 前言
最初因为公司业务需要，公司缺少测试，所以要求开发人员自行进行测试，甚至是自动化测试，各人负责自己的产品和功能。所以需要寻找一种比较合适的自动化测试框架来实现。

#### 框架选择
基于这几项标准：

* 同时支持iOS、Android、H5，且尽量能保持接口统一，减少开发维护成本；
* 编程语言支持Python/Ruby；
* 用户量大，文档丰富。

最后确定的是**Appium**

#### Appium简介
Appium的具体介绍，参考[Appium](http://appium.io/)。
这里是几点比较好的理念。

* 采用Appium时，无需对被测应用做任何修改，也无需嵌入任何东西；
* Appium对iOS和Android的原生自动化测试框架进行了封装，并提供了统一的API，减少了自动化测试代码的维护工作量；
* Appium采用Client-Server的架构设计，并采用标准的HTTP通信协议；Server端负责与iOS/Android原生测试框架交互，无需测试人员关注细节实现；Client端基本上可以采用任意主流编程语言编写测试用例，减少了学习成本。

### 环境准备（iOS）
#### 安装Appium依赖
    
验证：`$ appium-doctor --ios`
安装：`$ npm install appium-doctor -g`
再次验证：`$ appium-doctor --ios` 成功即可

![](https://ws3.sinaimg.cn/large/006tNc79gy1fmxmcbm2wzj30v60f2dk2.jpg)

#### 安装Appium的server部分

有多种方式安装：

* 源代码编译安装；
* Terminal中通过npm命令安装；
* 直接下载[appium.dmg](https://bitbucket.org/appium/appium.app/downloads/)安装

这里比较推荐第三个（我安装的是v1.7.1），直接安装应用程序，虽然看起来没有那么高大上，但是比较方便，除了GUI界面操作更直观以外，更重要的原因是，相比于命令行运行方式，Appium app多了一个Inspector模块，可以调用模拟器运行被测应用程序，并且可以更方便地在预览页面中查看UI元素的层级结构和详细控件属性，极大地提高了编写测试脚本的效率。为了操作更加方便，同时我也使用npm进行了安装。

提供相关命令：

```
>brew install node              //get node.js
>npm install -g appium          //get appium
>npm install wd             
```

#### 安装Appium的client部分
client部分实际就是安装脚本的编译环境。Appium最主流是使用ruby脚本或python脚本。刚开始我选择的是ruby脚本，在配置环境的过程中遇到了各种问题，耗时比较久，因为对ruby不够熟悉，最后还是选择了python来实现。（ruby的配置方法暂时不进行介绍）

配置python环境很简单：`$ pip install Appium-Python-Client` 

### Appium-desktop使用
先记住这个：
{
    "platformName": "iOS",
    "platformVersion": "11.0",
    "deviceName": "iPhone 7",
    **"automationName": "XCUITest",**
    "app": "/path/to/my.app"
}
很重要！很重要！很重要！

1. 启动Appium
2. start之后，点击放大镜图标进行配置，配置内容为上述的这个部分。
3. 点击“Start Session”
4. 等待片刻即开始运行。可以进行tap gesture sendKey 以及可以录制脚本（录制的脚本大多数情况下不能直接使用，会有一些方法的调整）。

### 开始python自动化测试
#### 编写python脚本
基于appium-desktop录制的脚本进行修改，得到如下脚本(login_test.py)：

```
import unittest
import os
from appium import webdriver
from time import sleep
from appium.webdriver.common.touch_action import TouchAction
from selenium.common.exceptions import NoSuchElementException


class  loginTest (unittest.TestCase):

	def  setUp(self):
		app = os.path.abspath('/Users/Jeffrey/Desktop/CareVoice.app')

		self.driver = webdriver.Remote(
			command_executor = 'http://127.0.0.1:4723/wd/hub',
			desired_capabilities = {
				'app':app,
                'platformName': 'iOS',
                'platformVersion': '11.2',
                'deviceName': 'iPad Air',
                'bundleId': 'co.kangyu.Kangyu',
                'automationName':'XCUITest',
                "noReset": 'true'
				# 'udid': '45751dc8cd8d737bbcea48a0307d50419161afb8'
			}
		)
	def test_login(self):
		print ("启动login测试")
		sleep(1)
		print ("开始测试")
		driver = self.driver

		el1 = driver.find_element_by_accessibility_id("Profile")
		el1.click()
		print ('点击profile')

		el2 = driver.find_element_by_accessibility_id("Sign In")
		el2.click()

		try:
			el3 = driver.find_element_by_xpath("//XCUIElementTypeApplication[@name=\"The CareVoice\"]/XCUIElementTypeWindow[1]/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeScrollView/XCUIElementTypeOther[1]/XCUIElementTypeScrollView")
		except NoSuchElementException:
			print ('已经登录了')
			return;
			# 执行退出登录
			TouchAction(driver).press(x=358, y=779).move_to(x=3, y=-303).release().perform()
			    
			TouchAction(driver).press(x=339, y=787).move_to(x=22, y=-272).release().perform()
			    
			TouchAction(driver).press(x=347, y=818).move_to(x=3, y=-444).release().perform()
			    
			ell1 = driver.find_element_by_accessibility_id("Settings")
			ell1.click()
			print ('点击设置')
			ell2 = driver.find_element_by_accessibility_id("Sign Out")
			ell2.click()
			print ('点击退出登录')

			el3 = driver.find_element_by_xpath("//XCUIElementTypeApplication[@name=\"The CareVoice\"]/XCUIElementTypeWindow[1]/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeScrollView/XCUIElementTypeOther[1]/XCUIElementTypeScrollView")
			el3.click()
		else:
			print ('还没有登录，所以可以找到')
			el3.click()

		print ('点击密码登录')
		el4 = driver.find_element_by_accessibility_id("phone")
		print ('点击用户名')
		el4.send_keys("13262851829")
		print ('输入用户名')

		el5 = driver.find_element_by_xpath("//XCUIElementTypeApplication[@name=\"The CareVoice\"]/XCUIElementTypeWindow[1]/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeScrollView")
		el5.click()
		print ('收起键盘')

		el6 = driver.find_element_by_xpath("//XCUIElementTypeApplication[@name=\"The CareVoice\"]/XCUIElementTypeWindow[1]/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeOther/XCUIElementTypeScrollView/XCUIElementTypeOther[3]/XCUIElementTypeOther[1]/XCUIElementTypeSecureTextField")
		el6.click()

		print ('点击密码')
		el6.send_keys("111111")
		print ('输入密码')

		# el5.click()
		# el7 = driver.find_element_by_accessibility_id("Sign In")
		# el7.click()
		# print ('点击登录')

		el8 = driver.find_element_by_accessibility_id("Return")
		el8.click()
		print ('点击登录')

		sleep(1)

		TouchAction(driver).press(x=394, y=303).move_to(x=-25, y=479).release().perform()

		el8 = driver.find_element_by_accessibility_id("Home")
		print ('点击Home')
		el8.click()

	def tearDown(self):
		sleep(5)
		print ('退出程序')
		self.driver.quit()

if __name__ == '__main__':
	suite = unittest.TestLoader().loadTestsFromTestCase(loginTest)
	unittest.TextTestRunner(verbosity=2).run(suite)

```

#### 开启Appium服务
`>appium &                       //start appium`

如果服务开启成功，则会出现这样：![](https://ws3.sinaimg.cn/large/006tNc79ly1fni9owc008j30us03i753.jpg)

#### 运行客户端脚本
`$ python3 login_test.py`

运行成功即会打开模拟器运行程序，可以添加一些输出语句，观察实际的操作过程。

### 补充
建议参考Appium的官方文档，有很大的参考价值。
这是Appium的[示例代码](https://github.com/appium/sample-code)。
python[示例](https://github.com/appium/python-client/tree/master/test/functional/ios)



