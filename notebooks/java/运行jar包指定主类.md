# 运行jar包指定主类

### 正常情况下，运行java程序

```shell
java test #运行 test.class文件，不要加后缀名即可运行
 
java -jar test.jar #运行 有入口类的可独立运行的 jar包
```

有入口类的jar包：包里的MANIFEST.MF文件中已设置了入口类的名字。
 入口类：即main函数所在的类。

### 运行java程序时，报错：“找不到或无法加载主类”

> 在intelliJ IDEA 下，工程结构已经指定了构建jar包的入口类的名称，**但实际上**生成的jar包中的清单文件MANIFEST.MF并没有Main-Class:这一行!
>  就是**没有入口类的**jar包。
>
> 



- 解决办法1：手工添加Main-Class

  用压缩工具打开jar文件 编辑**META-INF目录下的MANIFEST.MF文件** 保证前两行为:

  ```makefile
  Manifest-Version: 1.0
  Main-Class: MainClassName
  ```

  `MainClassName`应该改为你的工程的入口类名！

- 解决办法2: 运行时再指定入口类名

  jar包中没有入口类，用`-cp / --classpath`指定classpath

  ```shell
   java -cp HelloWorld.jar org.test.HelloWorld
  ```