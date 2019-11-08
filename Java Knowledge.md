### Java中的类加载器

#### java中常见的类加载器


#### 双亲委派类加载机制：

在java世界中，有一个双亲委派机制，是说要加载一个类的时候，先交给父级类加载器加载，父级类加载器无法加载时，自己才能去加载这个类。

![Alt text](https://github.com/Zeratal/Diaries-of-Phoenix-Daddy/blob/master/classloader.png)

这是我们在实现自己的类加载器时，普遍要遵守的一个机制。但是Ranger实现的类加载器破坏了双亲委派机制，从而将自己使用的第三方库和组件使用的第三方库隔离。由于ranger使用的第三方插件的所有类都是自己定义的类加载器加载的，因此不用担心第三方库冲突的问题。

![Alt text](https://github.com/Zeratal/Diaries-of-Phoenix-Daddy/blob/master/rangerloader.png)
