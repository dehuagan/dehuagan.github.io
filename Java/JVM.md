# Java内存区域

# 双亲委派机制
## 为什么需要双亲委派机制
首先，通过委派的方式，可以避免类的重复加载，当父加载器已经加载过某一个类时，子加载器就不会再重新加载这个类。

另外，通过双亲委派的方式，还保证了安全性。因为Bootstrap ClassLoader在加载的时候，只会加载JAVA_HOME中的jar包里面的类，如java.lang.Integer，那么这个类是不会被随意替换的，除非有人跑到你的机器上， 破坏你的JDK。

那么，就可以避免有人自定义一个有破坏功能的java.lang.Integer被加载。这样可以有效的防止核心Java API被篡改。


## loadClass过程
1、先检查类是否已经被加载过

2、若没有加载则调用父加载器的loadClass()方法进行加载

3、若父加载器为空则默认使用启动类加载器作为父加载器。

4、如果父类加载失败，抛出ClassNotFoundException异常后，再调用自己的findClass()方法进行加载。

## 破坏双亲委派
### why
因为在某些情况下父类加载器需要委托子类加载器去加载class文件。受到加载范围的限制，父类加载器无法加载到需要的文件，以Driver接口为例，由于Driver接口定义在jdk当中的，而其实现由各个数据库的服务商来提供，比如mysql的就写了MySQL Connector，那么问题就来了，DriverManager（也由jdk提供）要加载各个实现了Driver接口的实现类，然后进行管理，但是DriverManager由启动类加载器加载，只能记载JAVA_HOME的lib下文件，而其实现是由服务商提供的，由系统类加载器加载，这个时候就需要启动类加载器来委托子类来加载Driver实现，从而破坏了双亲委派，这里仅仅是举了破坏双亲委派的其中一个情况。

### 破坏双亲委派的实现
我们结合Driver来看一下在spi（Service Provider Inteface）中如何实现破坏双亲委派。

先从DriverManager开始看，平时我们通过DriverManager来获取数据库的Connection：

```js
String url = "jdbc:mysql://localhost:3306/xxx";
Connection conn = java.sql.DrivaerManager.getConnection(url,"root","root");
```

在调用DriverManager的时候，会先初始化类，调用其中的静态块：
```js
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}

private static void loadInitialDrivers () {
    ...
    // 加载Driver的实现类
    AccessController.doPrivileged(new PrivilegedAction<void>() {
        public void run() {
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();
            try{
                while(driversIterator.hasNext()){
                    driversIterator.next();
                }
            } catch(Throwable t){

            }
            return null;
        }
    });
    ...
}
```

重点来看一下ServiceLoader.load(Driver.class)：

```js
public static <S> ServiceLoader<S> load(Class<S> service) {
    //获取当前线程中的上下文类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

可以看到，load方法调用获取了当前线程中的上下文类加载器，那么上下文类加载器放的是什么加载器呢？

在sun.misc.Launcher中，我们找到了答案，在Launcher初始化的时候，会获取AppClassLoader，然后将其设置为上下文类加载器，而这个AppClassLoader，就是之前上文提到的系统类加载器Application ClassLoader，所以上下文类加载器默认情况下就是系统加载器。

![img](../img/loadclass.png)

# GC
[详细](https://mp.weixin.qq.com/s/_AKQs-xXDHlk84HbwKUzOw)



