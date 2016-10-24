## 译文

> 原文链接： [I set out to create pixel perfect icons. Here’s what I discovered along the way.](https://blog.ginetta.net/i-set-out-to-create-pixel-perfect-icons-heres-what-i-discovered-along-the-way-4e46378932df#.uoik0shhn)

### **我开始制作完美像素图标。以下是这一路探索所得。**

有些设计师对这种深入到像素的制作并不足够重视，其他人会认为它会魔性地从天而降，而只有极少数人接受这个挑战并切实地去创作。我个人深信不疑，图标是一个好的设计的非常重要的一环，因此，每个设计师都应该能够设计出一套图标系统，只要有热情、耐心和一些指导。

在 [Ginetta](http://www.ginetta.net/) 作为一个设计师，这一路上，我一直在寻找一篇能够教我如何进行图标设计，从文件配置到图标字体的完整的文章。我浏览过很多优秀的文章，全部都是不同的来源。就在某一时刻，我决定将所有这些信息收集起来并完善整理为一份指南，以此来帮助和鼓励设计师们开始去绘制出自己的完美像素图标。

这篇指南被分成五个部分，分别包括：配置文件、提示&奇淫技巧、图标网格、导出图标和创作图标字体。

### **第一部分：配置文件**

开始你图标设计之旅的第一步就是正确合适地配置好你的文件。这有助你于一开始就精确而有序。

![From left to right: Illustrator, Sketch and Affinity Designer.](http://7xtt0k.com1.z0.glb.clouddn.com/icon-tools.png)

绘制你的图标，会有不同的软件可供选择。最著名的几个当属 Adobe Illustrator、Sketch 和 Affinity Designer 了。我更偏爱于 AI，因为它提供了最简单有效的很多绘制工具（画笔工具、路径查找器、...），而这些都是在我绘制图标时对我最为重要的。[Scott Lewis](http://iconutopia.com/interview-scott-lewis-iconfinder/)，开发者 & Iconfinder 的内容负责人，曾在他的文章中比较过这三种软件。[Can Sketch or Affinity designer replace Adobe Illustrator?](Sketch 或者 Affinity designer 能够取代 Adobe Illustrator 吗？)：

> 我承认在多年的使用中会有偏见，但是用 Illustrator 的画笔工具绘制的感觉就像是在使用机械笔一样，而使用 Sketch 和 Affinity 的画笔工具的感觉则是像使用粉笔绘制一样。

我完全同意他的观点。

**创建新的文件**

在创建一个实例文件之前的第一件事就是要去定义一些关键的元素：

- 画板尺寸，宽度和高度应为偶数（不能用小数！）。

- 检查选项，将新的对象对齐到像素网格中，每次你新建一个图标，你绘制的对象将自动对齐到最近的像素。

- 如果你已经知道了你需要绘制几个图标，那么你也可以定义好画板数，间距和列。

- 你的高级设置应该会和下面的截图看起来相似。

![Creating a new document pop-up window.](http://7xtt0k.com1.z0.glb.clouddn.com/advanced-setting.png)

**通用配置**

在新建完一个文件后，会有一些小的调整需要设置：

- 打开 Illustrator 的 Edit（编辑） > Preferences（首选项） > General（常规） (或者按 Cmd + k 快捷键)

- 在 General（常规）选项中，将键盘增量设为 1 px。这样，当你移动你的图标时，它们将留在像素网格中。

![General settings pop-up window](http://7xtt0k.com1.z0.glb.clouddn.com/General-setting.png)

- 转到 Units（单位）选项卡，将 General（常规）和 Stroke（描边值）设为 Pixels（像素）。如果你愿意，你也可以将字体设为像素（这貌似非常没有必要，除非你真的打算在你的图标中使用字体）。

![Preferences pop-up window](http://7xtt0k.com1.z0.glb.clouddn.com/units-setting.png)

### **第二部分：提示和技巧**

现在你已经成功地创建了你的文件，是时候去浏览一些便捷的提示和技巧了。

#### **提示**

**提示 1：使用整数尺寸**

每当输入一个数字，请确保你是手动键入的，并且时刻谨记一定要使用整数，一定要使用整数（重要的事说三遍。。。不要用小数！）。为了确保你使用的是整数，尽量使用一些对象工具（椭圆，矩形等），而不是自己手工绘制。

**提示 2：所有的数字必须是偶数**

使用偶数是至为重要的，因为图标通常总是需要被调整大小。例如，假设你又一个 64 x 64 的图标，而你要将其调整为 32 x 32（一半），你仍将得到一个偶数而不是一个奇数或者是更糟糕的小数。因此，就不会有模糊不清的图标了！

**提示 3：时刻保持你的变换框为打开状态**

转换框（ Window（窗口） > Transform（变换）或者使用 shift + F8 快捷键）可以帮助你追踪那些意外的小数值。如果有，你可以立即输入最为接近的整数值来修正它。

![Screenshot of how your transform box should look like](http://7xtt0k.com1.z0.glb.clouddn.com/transform-box.png)

#### **技巧**

**技巧 1：开启像素预览**

有时，开启像素预览会很有帮助（在 View（视图） > Pixel Preview（像素预览）或使用 alt + cmd + y 快捷键开启设置）。这个预览模式会向你展示你图标的光栅图像，并且这将帮助你发现任何丢失的半个像素。这就是在你保存文件为 PNG 或者其它栅格格式之后所得到的图像。

**技巧 2：相信你的眼睛而不是数据**

在设计你的图标时，确保你时不时地将他们视为一组，一起显示在一块画板上。这有助于你找到那种平衡。或许他们是不一样的尺寸和形状，但最终他们会看起来好像就本该属于一起。

**技巧 3：使用标准的尺寸**

如果你的用户未要求特殊的尺寸，那就稳一点并使用标准的尺寸（例如：16 x 16, 48 x 48, 64 x 64, 92 x 92, 等）。如果你是为一些特殊的平台制作图标，像 Android 或者 IOS，查看其各自对应的指南（比如 Google 所推行的 Material Design）并遵循其标准的尺寸。这与一致性和缩放比例是相关的。

**技巧 4：从最大的尺寸开始设计**

当你设计图标时，我个人建议你从最大的尺寸开始设计，然后再缩小到其它尺寸。从大尺寸开始会给你更多的空间来绘制你的图标与你所想要的复杂度。接着你就可以缩放它，也可以根据你需求来调整和简化图标使其最为合适。在更大的图标里的一些细节到了小的图标就可能要简化了。

如果你在寻求一些灵感，可以到这些图标库里逛逛，比如 [The noun project](https://thenounproject.com/), [UI Parade](http://www.uiparade.com/skill-type/icons/), [Flat Icon](http://www.flaticon.com/), [Iconfinder](https://www.iconfinder.com/) 和 [Icons8](https://icons8.com/)。

### **第三部分：图标网格**

当我第一次开始制作图标时，我天真地以为在我还未尝试着画出任何一个图标之前我就需要一个网格。在一番挣扎和尝试过各种不同的方法后，我意识到对于你的第一个图标集真的不需要一个网格。事实上，在你考虑创建一个网格之前，你需要制作大量的图标以便于你能够真正理解和明白创建一个网格所需要考虑到的因素。

也就是说，当你绘制图标时使用网格系统通常来说是一个好的实践，因此，下面我将会解释什么是图标网格，什么时候使用图标网格以及怎么使用图标网格。

#### **什么是图标网格**

图标网格是一组规则，它定义了你构建的图标集的结构。这有助于保持你图标集的一致性。

![IOS 9 icon grid](http://7xtt0k.com1.z0.glb.clouddn.com/ios9-icon-grid.png)
参见 [IOS 9 设计指南](https://designcode.io/iosdesign-guidelines)

#### **何时又如何使用图标网格？**

当你可以应用网格系统时，这里有一些主要的应用实例：

- 你越是需要创造出更多风格一致的图标，参考指南就越是应该获得一个有凝聚力的集合以使其更为坚固。所以，如果你要制作一个大的图标集（50+），那就先画它几个图标（一个小样本），然后再决定图标网格的规则，并在之后的图标中都要坚守下去。

- 当为具有图标网格的现有平台（IOS ，Android，等）创建图标时，遵守其网格规范，以使你的图标能够和平台剩余部分保持一致。

- 定义指导规范是相当重要的，如果你知道将来会有其他人来接手这些图标。

> 所有伟大的图标集都有一个网格，但所有最糟的图标集也完全遵守于网格。 —— Justas，Icon Utopia

**规则：永远不要向你图标信息的清晰度妥协仅仅是因为它刚好适应了网格，因为你的网格不会适合每个图标**。如果一个图标不合适，就失去了其意义，或只是看起来丑死了，那么请不要害怕把网格给关掉。

如果你需要一些关于网格上的帮助，你可以看看这篇文章：[Icon Grid: When And How To Use It?](http://iconutopia.com/icon-grid-when-and-how-to-use-it-bonus-grids/)

### **第四部分：导出图标**

在完成你的图标制作的下一步就是导出这些图标，这样你就可以在你的网站或者应用上使用它们。这涉及到两个关键步骤，准备和保存文件。

#### **准备**

1. 在导出你的图标之前，**请务必确保在每个图标周围绘制边框，透明且与你的画板保持相同尺寸**。这将允许图标之间具有相同的距离，而不管它们的大小。

2. **不要直接从 Illustrator 将图标复制到 Photoshop 上**。它们总是会因此变的模糊不清。

3. 确保你的图标在它们各自本该在的画板上。

4. 选择你的画板（快捷键 shift + O），分别进入每个画板，并根据里面的图标去重命名它。

![Screenshot of how your icons should look like before you export them.](http://7xtt0k.com1.z0.glb.clouddn.com/icon-before-export.png)

#### **保存**

1. 找到 File（文件）> Save As（存储为，快捷键：cmd + shift + s），然后为你的图标输入一个简洁而有意义的名字。

2. 选择文件类型为 SVG 并选中 Use Artboards（使用画板）选项，这将会导出所有的图标。如果你想导出指定的图标，选中 Range（范围）选项，然后输入需要导出的画板编号。为什么选择 SVG 格式？为什么不是 EPS 格式？因为 SVG 是用于 Web 排版的格式，而至于 EPS 则是用于打印的发布环境。

3. 选择你想保存图标的路径然后点击保存按钮。

4. 最后，每个图标的文件名里都会有一个烦人的 “1-”。要删除它，右键单击这些图标，然后选择重命名所有，把 “1-” 替换为空字段。

![Screenshot of how your saving settings should look like before you export your icons.](http://7xtt0k.com1.z0.glb.clouddn.com/export-icon.png)

#### **优化**

不幸的是 Illustrator（甚至是 Sketch）在 web 上对图标的优化不是很友好。在你导出图标后，如果你用一些文本编辑器（如 Sublime 或者是 Atom）打开它们，你会注意到这些图标显示的代码会相当混乱。

这个在线工具 [svgomg](https://jakearchibald.github.io/svgomg/) 可以帮助你整理你的代码。只要拖动图标到上面并再次下载就可以了！你可以很直观的感受到它简化图标的代码是有多快，同时也将其文件大小减少至少 50%！

![Icon code before(left) and after(right) optimization.](http://7xtt0k.com1.z0.glb.clouddn.com/optimization-icon.png)

**如何生成所有尺寸的 PNG？**

有时候 SVG 格式将不会是正确的格式 —— 比如当你为本机移动端应用创作图标集时。在这种情况下，你可能需要导出 PNG。到 [Iconverticons](https://iconverticons.com/online/) 这个网站，拖放你的 SVG 图标，et voilà(哦，是的)！该应用会将它们转化为几乎是每一个可能的 PNG 分辨率图标。

### **第五部分：制作一个图标字体**

图标字体非常适合 Sketch，HTML 和其他工具中的设计原型。关于在开发网站时使用图标字体这一话题已经产生了[热烈的讨论](https://www.sitepoint.com/icon-fonts-vs-svg-debate/) —— 目前 SVG 仍然是最好的选择。Github 上最近也写了关于[这方面的东西](https://github.com/blog/2112-delivering-octicons-with-svg)。

这里有几个用来制作图标字体的选项，例如：

- [IcoMoon](https://icomoon.io/) —— 一个 web 应用，用来制作，编辑和分享图标字体。

- [Gulp Icon Font](https://github.com/nfroidure/gulp-iconfont) —— 一个命令行工具，可以从 SVG 图标的文件夹中生成字体。

- [Glyphs](https://glyphsapp.com/) —— 一个在 Mac 上的专业字体设计工具。它有一个更便宜的版本和生成图标字体所需的所有功能。

在这篇文章中，我们已经决定专注于第一个选项 —— IcoMoon。

#### **如何使用 IcoMoon 制作图标字体**

IcoMoon 是一个 web 应用，你无需用户账号也可以使用它。但是缺点就是，你制作的所有东西都将存储在你的浏览器中，除非你手动清除缓存。所以我建议你还是创建一个账号。

在创建完账号后，还有几个步骤你需要跟着做，以便让你的图标字体准备好。好啦，让我们开始做吧！

**编辑和准备**

你第一件需要做的事就是创建一个新的工程。在那之后，你需要导入你的图标，但在开始之前，还有几件事是你需要了解的：

- 确保你所有的笔画和任何文本都转换为轮廓（快捷键是 shift + cmd + O）。

- 你的图标资源应该要有相同的边框尺寸。IcoMoon 会自动检测图标的尺寸并且把图标对齐到基本网格，该网格应与你设计时的网格尺寸相匹配。

当所有图标都已导入进去，你就可以使用顶部的工具栏去编辑或者移除它们了。

![Toolbar of IcoMoon before you generate your iconfont.](http://7xtt0k.com1.z0.glb.clouddn.com/toolbar-before-generate-iconfont.png)

在编辑图标时，你可以加上一些标签以便可以在 IcoMon 或者 一个 HTML 图标浏览器中去搜索图标。

![Edit mode of an icon.](http://7xtt0k.com1.z0.glb.clouddn.com/edit-icon.png)

**生成**

在生成字体时，你会在生成面板中看到所有选中的图标带有各自的标签，名称和连字。

![Display of generated icons.](http://7xtt0k.com1.z0.glb.clouddn.com/display-of-generate-icon.png)

确保你的连字选项卡是开着的，因为它们是使用图标字体最快的方式，并允许你输入 “edit” 或 “account_circle” 来自动替换相应的图标（字形）。

![Toolbar of IcoMoon after you generate your iconfont.](http://7xtt0k.com1.z0.glb.clouddn.com/toolbar-after-generate-iconfont.png)

**下载**

在下载完图标字体之后，你会在下载文件夹下发现几个不同的文件：

![Example of a downloaded folder.](http://7xtt0k.com1.z0.glb.clouddn.com/example-of-download-folder.png)

- `fonts/`，包含其他格式的字体以在不同的 web 浏览器上使用。

- `fonts/[Font Name].ttf`，字体安装文件，以便在电脑中使用字体。

- `demo.html`，代表你的字体在网站上的显示效果的演示。

- `selection.json`，和你需要元数据（例如字体浏览器）或者重新导入字体到 IcoMoon 时相关联。

- `style.css`，在网站上使用字体时的渲染文件。

现在，你已经可以开始制作你自己的完美像素图标和生成你自己的图标字体了。如果你遵循这些建议，慢慢地耐心练习，我相信你很快就会创作出像专业级别的图标！

**受到鼓舞了吗？学习更多有关图标世界的东西。**

如果你愿意深入一探图标设计的世界，这里有一些很有价值的文章。其中让我最为鼓舞的一篇是 [How To Master Pixel Perfect Icons.](http://iconutopia.com/how-to-design-pixel-perfect-icons/)，其它相关的文章有：[7 Principles of Effective Icon Design](https://design.tutsplus.com/articles/7-principles-of-effective-icon-design--psd-147)，[How to design a top-quality icon](http://www.creativebloq.com/graphic-design/how-design-top-quality-icon-10135092)，[Comparing the Two Methods for Creating Line Icons: Offset Paths vs. Strokes](https://design.tutsplus.com/tutorials/comparing-the-two-methods-for-creating-line-icons-offset-paths-vs-strokes--cms-25134)，[The proper way of creating outline icons](http://iconutopia.com/proper-way-of-creating-outline-icons/) 和 [Icon Handbook](http://iconhandbook.co.uk/)。

最后，我要感谢我的同事 Jessica Goodson，Tibor Kranjc 和 Florian Schulz 帮助我完成此篇。

享受其中吧！

---

**参考文档**

> 什么是 Pixelperfect?

> PixelPerfect (完美像素) 指的是一个 UI 素材本身的像素对应屏幕上一个像素的情况，这种情况下 UI 素材映射到屏幕上时没有任何拉伸和压缩，这种情况下 UI 显示效果非常清晰完美。 —— [深入理解 Canvas Scaler](http://blog.sina.cn/dpool/blog/s/blog_4148e8630102vji9.html?vt=4)

- [Interview: Scott Lewis – Iconfinder](http://iconutopia.com/interview-scott-lewis-iconfinder/)

- [Adobe Illustrator 菜单项中英对照释义](http://tu.sioe.cn/illustrator/6407.html)

- [连字（Ligature）那些事儿](https://webzhao.me/ligatures.html)
