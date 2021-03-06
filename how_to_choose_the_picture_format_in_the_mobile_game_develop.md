[在手机游戏开发中如何选择图像素材格式？](http://zengrong.net/post/2070.htm)

How to choose the picture texture format in the game develop?

这是我在知乎上的一个回答，[这里是原文][2]。

回答的前提是：使用OpenGL来渲染。

分几个点来回答。

## 1. RGBA4444真的比RGBA8888占用的RAM要少

其实这里说的RAM，是指的显存而非内存。OpenGL支持以这几种形式来使用纹理资源(via <http://www.khronos.org/opengles/sdk/docs/man/xhtml/glTexImage2D.xml>)：

* `GL_UNSIGNED_BYTE（RGBA8888或RGBA888）`
* `GL_UNSIGNED_SHORT_5_6_5`
* `GL_UNSIGNED_SHORT_4_4_4_4`
* `GL_UNSIGNED_SHORT_5_5_5_1`

在程序将图片载入系统内存后，会根据你选择的形式（RGBA8888/RGB565 etc.）对其做一些处理（怎么处理后面说），然后就将这些纹理上传到显卡的显存，之后会把这些图片占用的内存删除掉。<!--more-->

也就是说，载入的图片在变成了显卡能处理的纹理之后，就根本不会保存在内存中，所以你看不到内存占用的变化。

当然，这个操作是由程序员自行控制（或者由你选择的框架来决定）的，你如果决定不删除它们而让它们留在内存中，那当然会占用系统内存。

## 2. TexturePackger导出的图片是个什么情况？

你可以拿一张图片尝试，在导出为RGBA8888和RGBA4444的时候，它们的文件大小确实是不同的。请看下面的图片，并注意我加亮的部分：

![RGBA8888](/wp-content/uploads/2014/04/rgba8888.png)

![RGBA4444](/wp-content/uploads/2014/04/rgba4444.png)

对于同一张图片，在RGBA8888格式下，唯一颜色数是5864；而RGBA4444格式下，唯一颜色数是1454，文件大小减少了80KB左右。

至于图片信息中显示的 Original Colors依然是32Bit，这是因为在图像处理软件显示图像的时候，内部使用的色彩是8888的。

## 3. 保存的文件是个什么情况？

上面的RGBA4444是否就真的使用的16bit（4x4）来保存每个像素呢？

不是。

我们知道，PNG格式可以保存成 8bit 和 24bit 以及 24bit(with alpha channel)=32bit 三种格式。而JPEG格式是24bit的。RGBA4444有16bit，所以无论如何是不能使用8bit格式来保存的。

因此，这个RGBA4444是使用24bit（或者32bit）格式来保存的。

那么，为什么RGBA4444的文件体积会比RGBA8888小呢？

注意上面的两个 Number of unique colors。在这里，压缩算法起了作用，将相同的颜色压缩了，导致文件体积变小。

不过，即使是采用同样的色深保存，我仍然要说的是，<u>RGBA4444比RGBA8888的图像质量会差一些</u>。

## 4. 怎么做到的？

要回答上面的下划线部分结论，我们需要提出一个新的问题：RGBA8888转换成RGBA4444，发生了什么变化？

先来看看它们分别代表什么：

* RGBA8888 : R 8bit + G 8bit + B 8bit + A 8bit
* RGBA4444 : R 4bit + G 4bit + B 8bit + A 8bit

8bit 能代表的最大数字是256，也就是说每种颜色可以表达256个级别，那么8x3=24bit（不算A）就能表现 2^{24} = 167772165 种颜色。

同样的，RGBA4444能表现的颜色是 2^{12} = 4096 种颜色。

也就是说，进行这种转换，是一定会丢失颜色信息的。

以 0xFFFFFFFF 这个RGBA8888 颜色为例，转换成 RGBA4444 可以这样做：

<pre lang="C++">
unsigned int pixel32 = 0xFFFFFFFF;
unsigned short pixel16 = 
            ((((pixel32 >> 0) & 0xFF) >> 4) << 12) | // R
            ((((pixel32 >> 8) & 0xFF) >> 4) <<  8) | // G
            ((((pixel32 >> 16) & 0xFF) >> 4) << 4) | // B
            ((((pixel32 >> 24) & 0xFF) >> 4) << 0);  // A
</pre>

最终的结果是 0xFFFF。

## 5. iOS用什么？

当然是用pvr格式。

pvr是iOS设备的图形芯片 [PowerVR 图形][4] 支持的专用压缩纹理格式。它在PowerVR图形芯片中效率极高，占用显存也小。
性能对比可以看这里：[In Depth iOS & Cocos2D Performance Analysis with Test Project][1]。

## 6. Android用什么？

Android设备就没有那么好的运气了。由于硬件平台不统一，每个厂商的GPU可能使用不同的纹理压缩格式。所以还是老老实实用PNG比较好。

## 7. 参考文章

* [PowerVR][3]
* [PowerVR 图形][4]
* [手持平台纹理格式说明][5]
* [拒绝忽悠 移动GPU全解读][6]
* [In Depth iOS & Cocos2D Performance Analysis with Test Project][1]


[1]: http://www.learn-cocos2d.com/2011/11/depth-ios-cocos2d-performance-analysis-test-project/#image-formats
[2]: http://www.zhihu.com/question/23256637/answer/24099601
[3]: http://en.wikipedia.org/wiki/PowerVR
[4]: http://www.imgtec.com/cn/powervr/powervr-graphics.asp 
[5]: http://wiki.c3.91.com/index.php?title=%E6%89%8B%E6%8C%81%E5%B9%B3%E5%8F%B0%E7%BA%B9%E7%90%86%E6%A0%BC%E5%BC%8F%E8%AF%B4%E6%98%8E
[6]: http://www.igao7.com/1218-vv-gpu.html
