>欲善其工必利其器,本篇介绍我们常用的工具android studio中gradle的一些常用配置

这里我先列举出这些常用的配置下面在一一细说
1.差异化配置buildConfigField
2.签名配置signingConfigs
3.多渠道打包
4.配置打包后生成apk名字

----------------------------------------------------------
0.常用小窍门
----------------------
在第一次启动AS时候往往会出现这个界面
![这里写图片描述](http://img.blog.csdn.net/20160814195641621)
并且一直卡在这进不去,很是烦恼..其实这是在检查你的 Android SDK但是由于墙的原因一直卡这.
解决办法
1.翻墙
2.在Android Studio安装目录下的 bin 目录下，找到 idea.properties 文件，在文件最后追加disable.android.first.run=true 。重新运行即可
</br>
</br>
github下载的项目没法运行
很多时候.gradle plugin版本和本地的as自带的版本不对应.然后刚好又被墙了.没有自备梯子.就导致无法编译.所以在open project之前对应自己本地as的版本来修改好是很重要的一件事.
首先我们可以新建个项目然后把下载的项目中gradle版本对应修改成本地已经存在的
需要修改的地方有两处
1.
![这里写图片描述](http://img.blog.csdn.net/20160814200416319)
2.
![这里写图片描述](http://img.blog.csdn.net/20160814200559838)
只需要将这两个地方的值换成你新建项目或者本地已经存在项目中对应的值.然后在同步

1.buildConfigFiel
-------------------
先列举一个场景，我们在App开发中通常都有测试服和正式服之分,因为host并不相同,那么发布时候很多人都是通过注释来切换,然而常在河边走哪有不湿鞋,即使小心翼翼在发布频率很高的时候也难免会弄错..buildConfigFiel配置正好能很好的解决这个问题.
配置非常简单如下
![这里写图片描述](http://img.blog.csdn.net/20160808221437372)
只需要在module目录下build.gradle中的buildTypes中添加

```
buildConfigField("boolean", "LOG_DEBUG", "false")
```
然后同步一下就会在 app/build/generated/source/BuildConfig/Build Varients/package name/BuildConfig目录下生成如下代码
![这里写图片描述](http://img.blog.csdn.net/20160808222724435)
然后我们就可以在代码中根据该属性控制服务器host切换或者log打印等等等
![这里写图片描述](http://img.blog.csdn.net/20160808223207281)

2.signingConfigs
-----------------------------------
在做地图或者一些需要正式打包的第三方功能的时候,打包调试修改在打包在调试在修改...,相信有不少小伙伴经历过这个苦不堪言的过程,那么只能说你对gradle还不了解,signingConfigs就可以很好的解决这个问题.
步骤如下
1.配置签名信息
![这里写图片描述](http://img.blog.csdn.net/20160809091320812)
点击OK后会在module的build.gradle目录下生成对应代码
![这里写图片描述](http://img.blog.csdn.net/20160809124824797)
2.配置buildType中签名
![这里写图片描述](http://img.blog.csdn.net/20160809125741384)
点击OK后会在module的build.gradle目录下生成对应代码
![这里写图片描述](http://img.blog.csdn.net/20160809125916632)
这个时候你在Run App就已经是正式签名的了 当然如果很熟悉gradle的语法上面的过程你也可以直接手写
但是到此是不是感觉有点不妥..因为签名的信息在build.gradle中是明文展示的.虽然不会打包到apk当中,但是会提交到代码仓库那么公司其他人就可以看到.所以为了安全我们新建一个signing.properties文件在module目录下然后将keyStore信息写在该文件中并且不提交到代码仓库.这样别人就无法看到了.
signing.properties配置如下
```
STORE_FILE=test.jks
STORE_PASSWORD=123456
KEY_ALIAS=test
KEY_PASSWORD=123456
```
![这里写图片描述](http://img.blog.csdn.net/20160810153315711)



这里需要注意下signing.properties中STORE_FILE签名文件的相对路径是相对app目录开始的即在这里是
C:\Users\MSI\Desktop\BaseDemo\app
在修改下signingConfigs中关于签名配置的部分先上效果图
![这里写图片描述](http://img.blog.csdn.net/20160810223007081)
在贴上代码方便复制

```
File propFile = file('signing.properties');
            if (propFile.exists()) {
                def Properties props = new Properties()
                props.load(new FileInputStream(propFile))
                if (props.containsKey('STORE_FILE') && props.containsKey('STORE_PASSWORD') &&
                        props.containsKey('KEY_ALIAS') && props.containsKey('KEY_PASSWORD')) {
                    storeFile = file(props['STORE_FILE'])
                    storePassword = props['STORE_PASSWORD']
                    keyAlias = props['KEY_ALIAS']
                    keyPassword = props['KEY_PASSWORD']
                }
            }
```

3.多渠道打包
-----------------------
由于国内众多的app下载平台为了方便不同平台app数据统计.所以多渠道打包就很有必要了.
1.AndroidManifest.xml中添加PlaceHolder

```
<!-- UMeng 配置-->
<meta-data android:value="${UMENG_CHANNEL_VALUE}" android:name="UMENG_CHANNEL"/>
```
截图如下
![这里写图片描述](http://img.blog.csdn.net/20160810225002548)

2.添加渠道
在app目录中build.gradle设置productFlavors 我这随便添加了2个
```
	productFlavors {
       xiaomi {}
       wandoujia {}
   }

   productFlavors.all {
       flavor -> flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
   }
```
效果如下
![这里写图片描述](http://img.blog.csdn.net/20160810230918474)

3.生成apk
敲下gradle assembleRelease指令就可以去喝咖啡了(记得配置gradle环境变量)
该命令是生成不同Product Flavor即不同渠道的release版本
除此之外 assemble 还能和 Product Flavor 结合创建新的任务，其实 assemble 是和 Build Variants 一起结合使用的，而 Build Variants = Build Type + Product Flavor ， 举个例子大家就明白了：

如果我们想打包wandoujia渠道的release版本，执行如下命令就好了：
gradle assembleWandoujiaRelease

如果我们只打wandoujia渠道版本，则：
gradle assembleWandoujia
此命令会生成wandoujia渠道的Release和Debug版本

同理我想打全部Release版本：
gradle assembleRelease
这条命令会把Product Flavor下的所有渠道的Release版本都打出来。

总之，assemble 命令创建task有如下用法：

**assemble**： 允许直接构建一个Variant版本，例如assembleFlavor1Debug。

**assemble**： 允许构建指定Build Type的所有APK，例如assembleDebug将会构建Flavor1Debug和Flavor2Debug两个Variant版本。

**assemble**： 允许构建指定flavor的所有APK，例如assembleFlavor1将会构建Flavor1Debug和Flavor1Release两个Variant版本。

当然还是建议直接手点不容易出错
![这里写图片描述](http://img.blog.csdn.net/20160814190537647)
生成的apk在module\build\outputs目录下
![这里写图片描述](http://img.blog.csdn.net/20160814191733378)

4.验证
验证代码如下

```
private String  getApplicationMetaValue(String name) {
        String value= "";
        try {
            ApplicationInfo appInfo =getPackageManager()
                    .getApplicationInfo(getPackageName(),
                            PackageManager.GET_META_DATA);
            value = appInfo.metaData.getString(name);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return value;
    }
```
通过该代码就可以获取到渠道名并打印出来,这里就不演示了

4.配置打包后生成apk名字
------------------------------
配置了多渠道打包后为了方便区分apk版本等等信息...我们在配置下生成APK的名字,只需在module/build.gradle文件下添加如下配置

```
def packageTime() {
    return new Date().format("yyyy-MM-dd", TimeZone.getTimeZone("UTC"))
}

android {
	//........
	applicationVariants.all { variant ->
        variant.outputs.each { output ->
            def outputFile = output.outputFile
            if (outputFile != null && outputFile.name.endsWith('.apk')) {
                File outputDirectory = new File(outputFile.parent);
                def fileName
                if (variant.buildType.name == "release") {
                    fileName = "app_v${defaultConfig.versionName}_${packageTime()}_${variant.productFlavors[0].name}_release.apk"
                } else if(variant.buildType.name == "debug") {
                    fileName = "app_v${defaultConfig.versionName}_${packageTime()}_${variant.productFlavors[0].name}_beta.apk"
                }
                output.outputFile = new File(outputDirectory, fileName)
            }
        }
    }
}
```
截图如下
![这里写图片描述](http://img.blog.csdn.net/20160814194929392)
这时候在生成APK就很容易区分了
![这里写图片描述](http://img.blog.csdn.net/20160814195256333)


细心的小伙伴应该发现在 app/build/outputs/apk 文件夹中，可以看到一些文件名末尾带有 unaligned 文字的冗余文件，这些文件没有用，可以删掉，只需要在上面的基础上在build.gradle 中添加下列命令就好了

```
applicationVariants.all { variant ->
        variant.outputs.each { output ->
            def outputFile = output.outputFile
            if (outputFile != null && outputFile.name.endsWith('.apk')) {
                File outputDirectory = new File(outputFile.parent);
                def fileName
                if (variant.buildType.name == "release") {
                    fileName = "app_v${defaultConfig.versionName}_${packageTime()}_${variant.productFlavors[0].name}_release.apk"
                } else if(variant.buildType.name == "debug") {
                    fileName = "app_v${defaultConfig.versionName}_${packageTime()}_${variant.productFlavors[0].name}_beta.apk"
                }
                output.outputFile = new File(outputDirectory, fileName)
            }


            // 删除unaligned apk
            if (output.zipAlign != null) {
                output.zipAlign.doLast {
                    output.zipAlign.inputFile.delete()
                }
            }
        }
    }
```
![这里写图片描述](http://img.blog.csdn.net/20160816222353843)
在生成apk就不在有unaligned文件了
![这里写图片描述](http://img.blog.csdn.net/20160816222716465)
