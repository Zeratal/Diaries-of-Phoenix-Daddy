### Java中的类加载器

java中常见的类加载器有

    启动（Bootstrap）类加载器：负责装载<Java_Home>/lib下面的核心类库或-Xbootclasspath选项指定的jar包。由native方法实现加载过程，程序无法直接获取到该类加载器，无法对其进行任何操作。 另外，java中有规定以java.*开头的类必须是BootstrapClassLoader来加载的。

    扩展（Extension）类加载器：扩展类加载器由sun.misc.Launcher.ExtClassLoader实现的。负责加载<Java_Home>/lib/ext或者由系统变量-Djava.ext.dir指定位置中的类库。程序可以访问并使用扩展类加载器。

    系统（System）类加载器：系统类加载器是由sun.misc.Launcher.AppClassLoader实现的，也叫应用程序类加载器。负责加载系统类路径-classpath或-Djava.class.path变量所指的目录下的类库。程序可以访问并使用系统类加载器。

### 双亲委派类加载机制：

在java世界中，有一个双亲委派机制，是说要加载一个类的时候，先交给父级类加载器加载，父级类加载器无法加载时，自己才能去加载这个类。



这是我们在实现自己的类加载器时，普遍要遵守的一个机制。但是Ranger实现的类加载器破坏了双亲委派机制，从而将自己使用的第三方库和组件使用的第三方库隔离。由于ranger使用的第三方插件的所有类都是自己定义的类加载器加载的，因此不用担心第三方库冲突的问题。

