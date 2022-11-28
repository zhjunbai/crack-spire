首先自己需要一个破解的spire，但是找了半天根本找不到，要么是没用的网站，要么是要付费，研究半天发现spire的破解其实非常容易，我都怀疑官方是不是没有做混淆，现在我演示一遍过程

首先关于工具，我们需要dnspy，rider（不是必须的）

rider需要授权，但是如果你注册个新账号就可以试用30天，已经足够了，rider的调试非常厉害，可以调试到dll库里面，所以没有rider的可以根据我最终的结论直接用dnspy修改ilcode

这篇文章首先破解spirepdf，因为spire doc的破解完全是另一种方法，所以可能会开另一篇文章讲spire doc的破解

首先先去spire官方下载spire.pdf，得到一个nuget包，我们可以解压它，得到dll

![在这里插入图片描述](https://img-blog.csdnimg.cn/9542162344c84dd28307d1fe4db90660.png)
这里我们先破解net6的，因为net4的破解没差别，只是名字变了

接着在rider里引用dll，创建一个20页的pdf，因为官方的限制是水印加10页转换限制

![在这里插入图片描述](https://img-blog.csdnimg.cn/33ab05f263ce440ca9c2e19923d52299.png)
这段代码就是spire官方提供的spirepdf创建pdf教程，链接在下面
[spirepdf创建pdf](https://www.e-iceblue.cn/document/create-a-new-pdf-document.html)
接着运行代码，创建pdf和其他类型文件
查看三个文件，发现都有水印，并且其他文件最多9页
![pdf](https://img-blog.csdnimg.cn/abbe824aa6b44bbf9edfe3e72f2881d7.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/b43a9c0b0c164cb5a492b926b282e336.png)
接着开始破解，首先在创建pdf文件对象创建处打上断点
![在这里插入图片描述](https://img-blog.csdnimg.cn/abbb81eedf0e419aabd5fd53189c58c4.png)
接着开始debug，逐过程直接跳过pdf对象的创建，查看pdf对象的数据
![在这里插入图片描述](https://img-blog.csdnimg.cn/d867719955ed47ea85dc96dced486823.png)
定位到pdfnewdocument对象
![在这里插入图片描述](https://img-blog.csdnimg.cn/29100b8a5bd44ee4bb704f9e3dad5015.png)
发现里面有个licenseprotector对象，看这个名字其实就能猜出大概了，这个类其实就是我们需要重点关注的对象，类型是spr加上一个类似徽章的符号

*其实你会发现上面还有个internal license字段，但是对他的修改很困难，因为它对应的是个枚举，在spire.pdf命名空间下，且这四个枚举名字都一样，我根本不知道怎么通过修改ilcode来绕过，后来我换了个方法，在后期检查license时，直接修改switch的值，因为switch的是枚举的int值，但是似乎四个switch路径进入的方法都是一样的，怎么改都没差别，所以这个方法直接放弃*

知道了这个对象在哪之后，我们就需要追踪它的创建过程，拿到类名

重新debug，进入pdfdocument的构造函数，逐过程几步后就会发现这行代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/56f783b653d147268caab71eef66dc0b.png)
我们点进pdfnewdocument类，直接搜索
![在这里插入图片描述](https://img-blog.csdnimg.cn/a3b9dbc2560949d9b11c0daa3c1c1d07.png)
发现this.licenseprotector对象，此时我们点进对象
![在这里插入图片描述](https://img-blog.csdnimg.cn/67c90cc9d6ec42f6b1cdecde8eb2422b.png)
看起来是new了一个对象，然后解析了internal license，我们点进这类，也就是spr\u2720

打开ilviewer，查看类

![在这里插入图片描述](https://img-blog.csdnimg.cn/9fcdf33c8e8a493bb9dc59a6bc16d41d.png)
其实这就是我们刚刚那个类，命名空间在spire.pdf里面
![在这里插入图片描述](https://img-blog.csdnimg.cn/67e6f8d46a634d9ea1e431c9091cbd71.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/155b65d86d25487da628007cac6d61a6.png)

我们还会会发现有一个bool返回值的方法，判断了四个enum，然后返回，这四个其实就是license，然后return了一个bool，那么很简单，我们直接在这里截胡，让他永远返回true

复制类名，打开dnspy

![在这里插入图片描述](https://img-blog.csdnimg.cn/f4b227857c2a4adc990145881cddfac4.png)
打开spire pdf这个dll，然后打开spire.pdf名称空间，或者直接搜索

![在这里插入图片描述](https://img-blog.csdnimg.cn/9781264f0183493488604ebf0f3e60b5.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/caf567f064814dacb51fe3540ce23a1e.png)
记住修改搜索范围为所选文件，然后搜索那个类名

似乎搜索不到，因为dnspy有些字符可以显示，有些不行，不行的会变成unicode的形式，所以这个字符也许可以显示，我们可以自己转换，我直接贴在这

spr✠

![在这里插入图片描述](https://img-blog.csdnimg.cn/9206f53e87894585ade8ca30516d91a0.png)
这样能搜索到了，dnspy这方面真奇怪

因为方法很少，所以我们可以直接找到那个方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/68b3c4e30e7b4287964253d0e47f33c7.png)
有趣的是，这个类最后有一堆byte，其实就是警告信息，长度应该是138

![在这里插入图片描述](https://img-blog.csdnimg.cn/e290a366cfd341749bd316a8495b1822.png)
回到正题，在bool方法的返回语句上右键，编辑il语句，记住一定要在return行上，不要放在上一行或者下一行，不然你不知道会跳到哪去

![在这里插入图片描述](https://img-blog.csdnimg.cn/5c646ca671ca4b28a25a3a618629bd48.png)

此时会跳到il编辑界面，当初你右键的位置会高亮显示

![在这里插入图片描述](https://img-blog.csdnimg.cn/5acf20cd3d864f4a9384557795047335.png)
这两行就是返回语句，我们把倒数第二行，也就是0090，修改为idc.i4.1，具体怎么改呢，你直接用鼠标点击ldloc.0，然后选择我说的那个指令就行了
![在这里插入图片描述](https://img-blog.csdnimg.cn/b09f0830c3964aab8393f039db8f29c8.png)


![在这里插入图片描述](https://img-blog.csdnimg.cn/019b2eadb4e24b3fb6fafe69ab5b02cb.png)
修改完之后保存，如果你没看见保存按钮，那是因为你屏幕缩放太大了，点击最上面窗口栏，双击最大化就行

![在这里插入图片描述](https://img-blog.csdnimg.cn/207d316ab95f47f89102b6f8fda65df1.png)
保存就是直接确定

![在这里插入图片描述](https://img-blog.csdnimg.cn/6f3bd347fe71408aa612add289e3604d.png)
我们可以看见代码已经变了，直接返回true
接着左上角，保存模块
![在这里插入图片描述](https://img-blog.csdnimg.cn/2fbdeba5c7564adcb8752dfe4926532b.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/75adea480637499f828611b48f583608.png)
直接确定，默认会保存到原文件，其中不应该出现任何警告提示报错，应该直接完成，如果出现了，你也许得重新来了

保存完毕之后，我们再测试一下我们的代码

![在这里插入图片描述](https://img-blog.csdnimg.cn/768d13d1969b44b590f24f65137a2e90.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/b87641cc1f13449db0fba546fb4412ea.png)
pdf文件没有水印了，其他文件格式也没有10页限制了，破解完成

下载链接在这，[github仓库](https://github.com/zhjunbai/crack-spire)
