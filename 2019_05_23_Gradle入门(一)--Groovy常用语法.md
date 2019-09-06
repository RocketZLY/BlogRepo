# Gradle入门(一)--Groovy常用语法

### 概述

Groovy是一种可以用于构建的DSL，基于Jvm支持所有的Java语法，同时又对Java进行了扩展，提供了大量的语法糖去简化我们的代码。在开发中既能像Java一样用面向对象的方式编写，也能像脚本一样用面向过程的方式编写。而本篇我们主要介绍Groovy在Gradle脚本中常用的语法，如果你想知道更多可以到官网查看[http://groovy-lang.org/](http://groovy-lang.org/)。

在开始之前你需要装好Java和Groovy环境，并配置好Intellij编译器。

0

### 前言

我们先解释下啥叫Groovy既能像Java一样用面向对象的方式编写，也能像脚本一样用面向过程的方式编写。

先看Groovy面向对象的方式

```groovy
class Test {

    static void main(String[] args) {
        println "do something"
    }
}
```

执行main方法会打印出do something，和Java一样编写一个个的类去完成代码。

在看看Groovy面向过程的方式

```groovy
println "do something"
```

通过groovy命令执行该文件会打印出do something，是不是感觉很爽，有种所见即所得的感觉。

接下来我们更深一步看看这个脚本为啥能执行，我们知道Groovy是基于Jvm的，那么这段代码肯定是编译成了一个class去执行的，然后我们将class反编译成java看看。

```java
public class Test extends Script {
    public Test() {
        CallSite[] var1 = $getCallSiteArray();
        super();
    }

    public Test(Binding context) {
        CallSite[] var2 = $getCallSiteArray();
        super(context);
    }

    public static void main(String... args) {
        CallSite[] var1 = $getCallSiteArray();
        var1[0].call(InvokerHelper.class, Test.class, args);
    }

    public Object run() {
        CallSite[] var1 = $getCallSiteArray();
        return var1[1].callCurrent(this, "do something");//打印do something
    }
}
```

你会发现虽然我们只写一行`println "do something"`但它帮我们包装成了一个类，真正执行是在run方法里面帮我们打印了do something。

那么稍作总结，Groovy既能像Java一样用面向对象的方式编写，也能像脚本一样用面向过程的方式编写，而脚本最终也是帮我们包装成类去执行。



### 变量的定义

对于变量的定义和Java大体相同

```groovy
int x = 10
double y = 3.14
```

稍有不同的是多了一种def定义方式，代表当前类型是可变的，根据使用的时候具体确定

```groovy
def x = 1
x = 1.1
x = "abc"
```



### 方法的定义

大体和Java相同，修饰符默认是public

```groovy
String test(){
    return "123"
}
```

不同的是可以通过def定义方法返回类型为可变的

```groovy
def test(){
    return "123"
}
```

还有点不同的是在有参方法调用的时候groovy可以省略大括号

```groovy
println("123")
println "123"
```

除此之外还有和kt很像的特性，当闭包是方法最后一个参数的时候，闭包可以写在括号外，而如果只有一个参数是闭包的话可以不用写大括号，至于闭包我们后面会说到

```groovy
def test1(String s, Closure closure) {

}
test1("1") {

}

def test2(Closure closure){
    
}
test2 {
    
}
```



### String

定义有三种方式

```groovy
def str1 = 'abc' //普通字符串
def str2 = '''
abc
def
''' //可以保留格式的字符串 比如换行
def str3 = "abc$str1" //可以使用占位符的字符串
```

占位符和kt完全一样，在字符串中想使用变量的只需要`$变量名`，如果是使用表达式的话`${表达式}`

```groovy
def str4 = "abc $str1"
def str5 = "abc: ${str1.length()}"
```



### 逻辑控制

先看最常用的for循环

```groovy
def sum = 0
for (i in 0..9) {
    sum += i
}

/*对List循环*/
for (i in [1, 2, 3, 4, 5, 6, 7, 8, 9]) {
    sum += 1
}

/*对Map进行循环*/
for (i in ["lili": 1, "luck": 2, "xiaoming": 3]) {
    sum += i.value
}
```

至于list和map后面马上会讲，然后再是Groovy对switch做了很大的增强

```groovy
def x = 1.23
def result
switch (x) {
    case "foo":
        result = "found foo"
        break
    case "bar":
        result = "bar"
        break
    case [1.23, 4, 5, 6, "inList"]://列表
        result = "list"
        break
    case 12..30://范围
        result = "range"
        break
    case Integer:
        result = "int"
        break
    case BigDecimal:
        result = "bigDecimal"
        break
    default:
        result = "default"
        break
}
```

基本上你能想到的类型都可以作为case判断的条件



### List

定义方式如下

```groovy
def list = [-3, 9, 6, 2, -7, 1, 5]
```

你可能会问这不是和Java的数组冲突了，那要想定义一个数组怎么搞呢

```groovy
int[] array = [-3, 9, 6, 2, -7, 1, 5]
def array = [-3, 9, 6, 2, -7, 1, 5] as int[]
```

上面两种方式都可以定义数组，但是其实没有必要，数组的功能List都已经包含了，至于List的取值可以通过如下方式获取元素

```groovy
def list = [-3, 9, 6, 2, -7, 1, 5]
list[0]
```



### Map

定义方式如下

```groovy
def colors = [red: 'ff0000', green: '00ff00', blue: '0000ff']
```

有个需要注意的点是map的key全部都为String，即使我这里没有显示声明为String但都会默认转为String

至于获取元素有两种方式

```groovy
colors['red']
colors.red
```

添加的话除了常用的put()也有简化的方式

```groovy
colors.yellow = 'ffff00'
```



### Range

这个是Groovy特有的数据结构范围，定义比较简单

```groov
def range = 1..10
```

大部分时候都是用在for循环或者swtich的case中判断在不在这个范围



### Closure

闭包就是一段封闭的代码块定义如下

```groovy
def Closure = {
	
}
```

调用的话有两种方式，我个人更倾向使用第二种方式

```groovy
closure.call()
closure()
```

一般情况下闭包都是作为方法参数而使用，以map的each方法来举例

```groovy
def map = [1: "aaa", 2: "bbb", 3: "ccc"]
map.each {
    println "key: $it.key,value: $it.value"
}
```

该方法会打印出map的key和value，不过对于上面这个你肯定会有个疑问it是啥，它是当闭包只有一个参数的时候我们可以不用显式的声明，默认会提供一个it变量代表那个唯一的参数。那么你可能又会问那我怎么知道闭包中的参数呢，这个就得看map的each方法了

```groovy
    /**
     * Allows a Map to be iterated through using a closure. If the
     * closure takes one parameter then it will be passed the Map.Entry
     * otherwise if the closure takes two parameters then it will be
     * passed the key and the value.
     * <pre class="groovyTestCase">def result = ""
     * [a:1, b:3].each { key, value -> result += "$key$value" }
     * assert result == "a1b3"</pre>
     * <pre class="groovyTestCase">def result = ""
     * [a:1, b:3].each { entry -> result += entry }
     * assert result == "a=1b=3"</pre>
     *
     * In general, the order in which the map contents are processed
     * cannot be guaranteed. In practise, specialized forms of Map,
     * e.g. a TreeMap will have its contents processed according to
     * the natural ordering of the map.
     *
     * @param self    the map over which we iterate
     * @param closure the 1 or 2 arg closure applied on each entry of the map
     * @return returns the self parameter
     * @since 1.5.0
     */
    public static <K, V> Map<K, V> each(Map<K, V> self, @ClosureParams(MapEntryOrKeyValue.class) Closure closure) {
        for (Map.Entry entry : self.entrySet()) {
            callClosureForMapEntry(closure, entry);
        }
        return self;
    }
```

注释里面可以看到closure支持1或者2个参数，那么参数类型究竟是啥呢我们接着看`callClosureForMapEntry()`

```groovy
    protected static <T, K, V> T callClosureForMapEntry(@ClosureParams(value=FromString.class, options={"K,V","Map.Entry<K,V>"}) Closure<T> closure, Map.Entry<K,V> entry) {
        if (closure.getMaximumNumberOfParameters() == 2) {
            return closure.call(entry.getKey(), entry.getValue());
        }
        return closure.call(entry);
    }
```

这里就很明显了当闭包为两个参数的时候，参数分别是key和value，为一个参数的时候就是map的entry对象。所以当我们不清楚闭包的参数的时候，要么去找文档，要么去查看源码即可知道。

对于闭包的默认参数it需要着重说下，虽然它和kt的lambda表达式很像但是也有不同，即我们自己写的闭包没有定义参数它都会默认有个it参数，直接看例子

```groovy
def closure = {
    println it
}
closure("abc")//会打印出abc
```



### 总结

上面介绍的是Groovy在写Gradle脚本中常用的语法，当然Groovy语法并不止这么点，也有很像kt一样的扩展方法，委托等等一些高级用法不过在脚本中我们基本用不到也就不介绍了。
















