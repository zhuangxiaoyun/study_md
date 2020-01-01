# gradle

## 1、gradle与groovy基础

1. gradle wrapper
   1. 进入项目根目录
   2. 执行“gradle wrapper”
   3. 会在项目的根目录中生成一个文件夹
      1. ![1577610055431](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577610055431.png)
      2. 用途：用来下载正真的gradle【在项目拷贝到其他机器的时候，项目会先下载匹配的gradle】
2. gradle3.0之后使用daemon模式
   1. client jvm：接受请求和转发请求
   2. daemon jvm：在请求处理完成后，client jvm会被销毁，daemon jvm不会被销毁daemon jvm中可缓存之前处理请求加载的jar包
      1. daemon jvm 3小时不使用，则会被销毁
   3. client jvm和daemon jvm 的兼容性
      1. 版本不同，则会新建一个daemon jvm
3. ./gradlew compile java
   1. 判断服务器上是否有对于的gradle应用，如果没有则下载
   2. 查找对应的gradle版本的daemon jvm进程，如果没有找到，则新建一个daemon jvm，否则连接这个daemon jvm

## 2、gradle构建

1. gradle生命周期

   1. 初始化
   2. configuration
   3. execution

2. ```groovy
   task("first"){
       print("configuration")
       doLast {
           print("execution task")
       }
   }
   ```

   

## 3、插件编写入门

1. 编写在build.gradle文件中

   ```
   // 引入插件，实际上是调用project的apply()方法
   apply([plugin: 'java'])
   
   //groovy中在参数为map的时候，可以省略[]
   apply(plugin: 'java')
   
   // groovy在不引起歧义的时候，可以省略()
   apply plugin: 'java'
   ```

   

   ```groovy
   class MyPlugin implements Plugin<Project> {
       @Override
       void apply(Project project) {
           (0..<10).each { i ->
               project.task('task' + i) {
                   doLast {
                       print("execute task" + i)
                   }
               }
           }
       }
   }
   apply plugin: MyPlugin
   ```

2. script插件

   ```
   apply plugin: 'http://my_script.gradle'
   ```

3. binary插件

   1. gradle build 包含了3个部分

      1. 初始化

      2. configuration

         1. 会把build.gradle从上到下执行一次

         2. buildscript

            1. ```groovy
               buildscript {
                   repositories {
                       mavenCentral()
                   }
                   dependencies {
                       classpath group: 'org.apache.commons', name: 'commons-lang3', version: '3.9'
                   }
               }
               ```

               

      3. execution

         1. compile java

      4. configuration和execution使用classpath是不同的，

         1. configuration使用的是buildscript中定义的dependencies
         2. execution使用的是外层的dependencies

4. buildSrc插件

   1. 在build.gradle文件外层如果发现存在buildSrc项目，则会先编译buildSrc项目，然后把编译结果buildSrc.jar放到buildscript的classpath中
   2. ![1577607820897](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577607820897.png)

5. 发布的插件