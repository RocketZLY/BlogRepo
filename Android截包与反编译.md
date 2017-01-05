近期由于学习需要,尝试了下截包与反编译,发现对于简单的反编译与截包其实挺简单的,而本文也主要介绍截包与反编译工具的使用.

**截包**
------
截包的工具有很多,我这里主要介绍简单实用的fiddler.
官网下载地址:http://www.telerik.com/fiddler
下载完成后打开fiddler

1.首先，确保安装 Fiddler 的电脑和你的手机在同一局域网内，因为Fiddler只是一个代理，需要将手机的代理指向 PC 机，不能互相访问是不行的。

2.打开菜单栏中的Tools>Fiddler Options，打开“Fiddler Options”对话框。

![](http://img.blog.csdn.net/20160324215341903)

在Fiddler Options”对话框切换到“Connections”选项卡，然后勾选“Allow romote computers to connect”后面的复选框，然后点击“OK”按钮,设置允许远程链接.

![](http://img.blog.csdn.net/20160324215515628)

3.然后在本机命令行输入ipconfig命令行找到本机ip,由于我这里手机和电脑都是连得无线网所以使用电脑无线网ip 192.168.0.102

![](http://img.blog.csdn.net/20160324215906264)

4.打开手机设置无线网代理界面 服务器设置为电脑无线网ip 端口为fiddler监听端口8888

![](http://img.blog.csdn.net/20160324223101074)

5.现在就可以抓包了打开fiddler

![](http://img.blog.csdn.net/20160324223237003)

6还可以设置过滤器 过滤掉不想要的信息

![](http://img.blog.csdn.net/20160324223700384)

**反编译**
------
apktool : 它可以解码资源接近原始形式和重建后做一些修改,可以提取出图片文件和布局文件进行使用查看
下载链接:http://download.csdn.net/detail/zly921112/9472996

dex2jar : 将apk反编译成java源码(classes.dex转化成jar文件）
下载链接:http://download.csdn.net/detail/zly921112/9473310

jd-gui : 查看用dex2jar转换生成的jar文件,查看源码
下载链接:http://download.csdn.net/detail/zly921112/9473311

**apktool(反编译资源文件)**

将工具中apktool,解压得到aapt.exe，apktool.bat，apktool.jar,将需要反编译的APK文件放到该目录下，然后将命令行目录切换至该目录下.输入apktool d test.apk,命令中test.apk指的是要反编译的APK文件全名，反编译后生成文件名与APK文件名相同

![](http://img.blog.csdn.net/20160326164050100)

虽然这里有部分资源未能解码但是大部分资源都得到了,

![](http://img.blog.csdn.net/20160326164343198)

test文件内部

![](http://img.blog.csdn.net/20160326164428824)

如果你想将反编译完成的文件重新打包apk,那你可以输入 apktool b test(编译出来文件夹):

![](http://img.blog.csdn.net/20160326165918593)

在test文件下会发现多了两个文件 build 和 dist(里面存放着打包出来的APK文件)

![](http://img.blog.csdn.net/20160326170050664)

关于命令官网有给出例子
解码:

![](http://img.blog.csdn.net/20160326170445992)

构建:

![](http://img.blog.csdn.net/20160326170622401)

 **dex2jar+jd-gui(反编译.dex文件获取java源码)**

 将要反编译的apk后缀改成.rar 解压,得到其中的额classes.dex文件（它就是java文件编译再通过dx工具打包而成的），将获取到的classes.dex放到之前解压出来的工具dex2jar-0.0.9.15文件夹内，然后将命令行目录切换至该目录下.输入dex2jar.bat   classes.dex

![](http://img.blog.csdn.net/20160326172818896)

在该目录下会生成一个classes_dex2jar.jar的文件，然后用jd-gui文件夹里的jd-gui.exe，打开之前生成的classes_dex2jar.jar文件，便可以看到源码了，效果如下：

![](http://img.blog.csdn.net/20160326173117682)

由于源码被混淆了所以只能看到abc这种,其实挺恶心的,我还专门去查了反混淆希望能还原但是未发现解决办法,如果有朋友能反混淆欢迎交流谢谢.
