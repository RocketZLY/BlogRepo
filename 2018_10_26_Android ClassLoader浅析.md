# Android ClassLoader浅析

### 前言

最近在看Tinker的原理，发现核心是通过`ClassLoader`做的，由于之前也从未接触过`ClassLoader`趁着上周末看了安卓`ClassLoader`相关源码，这里分享一发安卓的`ClassLoader`和热更新的实现原理。

### ClassLoader

首先我们要知道，程序在运行时要把对应的类加载到内存，在安卓上来说就是把Dex文件中的类加载到内存，这个加载流程是通过`ClassLoader`实现的。因此如果我们想要动态加载自己的类，就得从`ClassLoader`上做文章，那么接下我们先看看安卓的`ClassLoader`。

```java
public abstract class ClassLoader {
    private ClassLoader(Void unused, ClassLoader parent) {
        this.parent = parent;
    }
    
    protected ClassLoader(ClassLoader parent) {
        this(checkCreateClassLoader(), parent);
    }
    
    protected ClassLoader() {
        this(checkCreateClassLoader(), getSystemClassLoader());
    }
    
    public static ClassLoader getSystemClassLoader() {
        return SystemClassLoader.loader;
    }
    
    static private class SystemClassLoader {
        public static ClassLoader loader = ClassLoader.createSystemClassLoader();
    }
    
    private static ClassLoader createSystemClassLoader() {
        String classPath = System.getProperty("java.class.path", ".");
        String librarySearchPath = System.getProperty("java.library.path", "");

        return new PathClassLoader(classPath, librarySearchPath, BootClassLoader.getInstance());
    }
    
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
    
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
}
```

可以看到在构造方法的时候我们需要传入一个父`ClassLoader`，默认没传情况下是以`PathClassLoader`作为父加载器的。

然后加载类是调用的`loadClass()`方法，他是先判断是本`ClassLoader`是否已经加载过，如果没加载过就让父加载器加载，然后父加载器看自己是否加载过，如果没加载过让祖父加载器加载，然后祖父加载器没找到，父加载器在找，父加载器没找到，子加载器再找，这也就是我们常说的双亲委派模式。

采取双亲委派模式主要有两点好处： 
1. 避免重复加载，如果已经加载过一次Class，就不需要再次加载，而是先从缓存中直接读取。 
2. 更加安全，如果不使用双亲委托模式，就可以自定义一个String类来替代系统的String类，这显然会造成安全隐患，采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载，也就无法自定义String类来替代系统的String类，除非我们修改类加载器搜索类的默认算法。还有一点，只有两个类名一致并且被同一个类加载器加载的类，Java虚拟机才会认为它们是同一个类。

那么下面的重点是看Android的类加载器如何重写`findClass()`方法来加载类的。

### Android ClassLoader继承关系

![](http://rocketzly.androider.top/classloader_architecture.png)

从上图可以看到与`ClassLoader`相关的一共有7个，大致可以分为如下3类

- `BootClassLoader`是`ClassLoader`内部类，是Android中所有`ClassLoader`的parent，可见性为包内可见我们无法使用。
- `BaseDexClassLoader`是`PathClassLoader`、`DexClassLoader`、`InMemoryDexClassLoader`的父类，类加载的主要逻辑都是在`BaseDexClassLoader`完成的。
- `SecureClassLoader`继承了抽象类`ClassLoader`，拓展了`ClassLoader`类加入了权限方面的功能，加强了安全性，其子类`URLClassLoader`是用URL路径从jar文件中加载类和资源，我们用不上。



我们重点关注与我们热更新相关的`PathClassLoader`和`DexClassLoader`。

- `PathClassLoader`只能加载已经安装到Android系统中的apk文件（/data/app目录），是Android默认使用的类加载器。
- `DexClassLoader`可以加载任意目录下的dex/jar/apk/zip文件，比PathClassLoader更灵活，是实现热修复的重点。

这里我们可以验证下`PathClassLoader`是安卓默认使用的类加载器。在`Activity`的`onCreate()`方法中打印它的类加载器。

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Log.i("zhuliyuan", this.getClass().getClassLoader().toString());
    }
```

![](http://rocketzly.androider.top/%E5%AE%89%E5%8D%93%E9%BB%98%E8%AE%A4classloader.png)



### 源码查看（基于API28）

```java
/**
 * Provides a simple {@link ClassLoader} implementation that operates on a list
 * of files and directories in the local file system, but does not attempt to
 * load classes from the network. Android uses this class for its system class
 * loader and for its application class loader(s).
 */
public class PathClassLoader extends BaseDexClassLoader {
    
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent){
        super(dexPath, null, librarySearchPath, parent);
   }
}

/**
 * A class loader that loads classes from {@code .jar} and {@code .apk} files
 * containing a {@code classes.dex} entry. This can be used to execute code not
 * installed as part of an application.
*/
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
                          String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}

```

从注释上可以看出`PathClassLoader`是用于系统类和应用程序类的加载，`DexClassLoader`可以用来加载任意目录的dex。具体实现还得看`BaseDexClassLoader`的构造方法。

```java
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
                              String librarySearchPath, ClassLoader parent) {
        this(dexPath, optimizedDirectory, librarySearchPath, parent, false);
    }

    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
                              String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
        ...
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        ...
        Class c = pathList.findClass(name, suppressedExceptions);
        ...
        return c;
    }
    
    @Override
    protected URL findResource(String name) {
        return pathList.findResource(name);
    }
    
    @Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }
}
```

可以看出在构造方法中创建了一个`DexPathList`对象赋值给了`pathList`字段，然后`findxxx()`方法都是从`DexPathList`中查找。

`BaseDexClassLoader`的构造函数包含四个参数：

- dexPath：包含类和资源的jar / apk文件列表，由 `File.pathSeparator`分隔，在Android上默认为：。
- optimizedDirectory：由于dex文件被包含在APK或者Jar文件中,因此在装载目标类之前需要先从APK或Jar文件中解压出dex文件,该参数就是制定解压出的dex 文件存放的路径。这也是对apk中dex根据平台进行ODEX优化的过程。自API26开始无效。
- librarySearchPath：指目标类中所使用的C/C++库存放的路径，可以为null。
- parent：父`ClassLoader`引用。

接下来我们查看`DexPathList`的构造方法和`findxxx()`方法。

```java
final class DexPathList {
	private Element[] dexElements;
    
    DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
        ...
        // 加载dexPath路径下的dex和resource
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);
        ...
    }
    
        private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
            List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
      Element[] elements = new Element[files.size()];
      int elementsPos = 0;
      for (File file : files) {
          if (file.isDirectory()) {
              elements[elementsPos++] = new Element(file);
          } else if (file.isFile()) {
              String name = file.getName();

              DexFile dex = null;
              if (name.endsWith(DEX_SUFFIX)) {
                  // Raw dex file (not inside a zip/jar).
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      System.logE("Unable to load dex file: " + file, suppressed);
                      suppressedExceptions.add(suppressed);
                  }
              } else {
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                  } catch (IOException suppressed) {
                      suppressedExceptions.add(suppressed);
                  }

                  if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
          } else {
              System.logW("ClassLoader referenced unknown path: " + file);
          }
      }
      return elements;
    }
}
```

构造方法中调用`makeDexElements()`方法获取到了`Element[]`数组赋值给了`dexElements`变量。

```java
    public Class<?> findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }

        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```

而`findClass()`方法则是变量在构造方法初始化好的`Element[]`数组中从前往后遍历找到我们需要的类。



这里我们总结下类加载的过程

- `PathClassLoader`和`DexClassLoader`调用了父类`BaseDexClassLoader`的构造方法。
- `BaseDexClassLoader`在构造方法中创建了`DexPathList`对象并赋值给`pathList`字段，加载类的`findxxx()`方法都是调用`DexPathList`类的`findxxx()`方法来实现内的加载。
- `DexPathList`在构造方法中调用`makeDexElements()`方法创建了`Element[]`数组赋值给`dexElements`字段，`findClass()`方法就是从前往后遍历`Element[]`数组找到我们要的class.

### 热更新实现原理

上面已经分析了类的加载流程，那么要想实现热更新我们需要拿到`PathClassLoader`中的`DexPathList`对象然后在拿到他当中的`Element[]`数组将我们已经修复的`Element`插入到数组的前面，这样在类加载时候就会先拿到我们已经修复的class了达到动态替换的效果。

加载我们自己的dex文件需要用到`DexClassLoader`，然后通过反射取出`DexClassLoader`中的`DexPathList`对象中的`Element[]`数组插入到`PathClassLoader`中`Element[]`数组前面即可。

热更新实际例子我会在另一篇[Android 手动实现热更新](https://blog.csdn.net/zly921112/article/details/83547750)介绍。





