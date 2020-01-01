scala

# 1、安装

## window 上安装 Scala

### 第一步：Java 设置

检测方法前文已说明，这里不再描述。

如果还未安装，可以参考我们的[Java 开发环境配置](https://www.runoob.com/java/java-environment-setup.html)。

接下来，我们可以从 Scala 官网地址 http://www.scala-lang.org/downloads 下载 Scala 二进制包(页面底部)，本教程我们将下载 *2.11.7*版本，如下图所示：

![img](https://www.runoob.com/wp-content/uploads/2015/12/FFFE6958-C5EB-4C7A-B720-8432B770428D.jpg)

下载后，双击 msi 文件，一步步安装即可，安装过程你可以使用默认的安装目录。

安装好scala后，系统会自动提示，单击 finish，完成安装。

右击我的电脑，单击"属性"，进入如图所示页面。下面开始配置环境变量，右击【我的电脑】--【属性】--【高级系统设置】--【环境变量】，如图：

![img](https://www.runoob.com/wp-content/uploads/2015/12/scala1.jpg)

设置 SCALA_HOME 变量：单击新建，在变量名栏输入：**SCALA_HOME**: 变量值一栏输入：**D:\Program Files（x86）\scala** 也就是 Scala 的安装目录，根据个人情况有所不同，如果安装在 C 盘，将 D 改成 C 即可。

![img](https://www.runoob.com/wp-content/uploads/2015/12/092523_lmSn_569074.png)

设置 Path 变量：找到系统变量下的"Path"如图，单击编辑。在"变量值"一栏的最前面添加如下的路径： %SCALA_HOME%\bin;%SCALA_HOME%\jre\bin;

**注意：**后面的分号 **；** 不要漏掉。

![img](https://www.runoob.com/wp-content/uploads/2015/12/scala3.jpg)

设置 Classpath 变量：找到找到系统变量下的"Classpath"如图，单击编辑，如没有，则单击"新建":

- "变量名"：ClassPath
- "变量值"：.;%SCALA_HOME%\bin;%SCALA_HOME%\lib\dt.jar;%SCALA_HOME%\lib\tools.jar.;

**注意：**"变量值"最前面的 .; 不要漏掉。最后单击确定即可。

![img](https://www.runoob.com/wp-content/uploads/2015/12/scala4.jpg)

检查环境变量是否设置好了：调出"cmd"检查。单击 【开始】，在输入框中输入cmd，然后"回车"，输入 scala，然后回车，如环境变量设置ok，你应该能看到这些信息。

![img](https://www.runoob.com/wp-content/uploads/2015/12/scala5.jpg)

## idea配置scala插件

![1574570222241](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1574570222241.png)

# 2、快速入门

## 1、示例

![1574579524005](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1574579524005.png)

函数说明：

![1574579548136](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1574579548136.png)

1. 使用scalac将scala源码编译成字节码
2. 使用scala运行字节码
3. 直接使用scala可以直接运行scala源码，但是速度慢，其底层也是先转成字节码后执行

```scala
// 1、object TestScala对应的是一个TestScala$的一个静态对象 MODULE$
// 2、且是一个单例对象
```

## 2、scala注意事项

- scala源文件以“.scala”为扩展名
- 执行入口是main()函数
- 严格区分大小写
- 每条语句不需要分号

## 3、scala字符串输出的三种方式

- 字符串通过+号连接
- **printf**
- **$引用变量**

```scala
// 1、object TestScala对应的是一个TestScala$的一个静态对象 MODULE$
// 2、且是一个单例对象
object TestScala {
  def main(args: Array[String]): Unit = {
    var str1: String = "hello"
    var str2: String = "world"
    println(str1 + str2)

    var name: String = "zs"
    var age: Int = 10
    var sal: Float = 16.67f
    var height: Double = 180.11
    printf("名字=%s,年龄=%d,薪水=%.2f,身高=%.3f", name, age, sal, height)
    
//    scala支持使用$输出内容
    println(s"个人信息：\n 名字：$name\n age:$age\n 薪水：$sal")
//    如果字符串中出现了类似${age+10}则表示{}是一个表达式
    println(s"个人信息2：\n 名字：${name}\n age:${age+10}\n 薪水：$sal")
  }
}
```

在scala中，整数默认是int，小数默认是double

逃逸分析：生命周期长且会被多次引用则放到堆里，不然放到栈里【放在堆的原因：多个栈之间可以共用数据，一个方法被调用就会创建一个栈】

## 4、类型推导

**val |var 变量名[:变量类型]=变量值**

1. 声明变量时，类型可以省略【编译器自动推导】
2. 类型确定后，就不能修改【说明scala是强数据类型语言】
3. var修饰的变量可以改变，val修饰的变量不可修改
4. val修饰的变量在编译后，等同于加上了final

val：没有线程安全问题，效率高，scala作者推荐使用val【val声明的变量可以修改属性，但是不能改变对象本身】

# 3、数据类型

## 1、简介

![1575082040242](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575082040242.png)

1. scala中数据类型都是**对象**【即scala没有java中的原生类型】
2. scala数据类型分为两大类：**AnyVal**(值类型)和**AnyRef**(引用类型)【不管是值类型还是引用类型都是对象】
   1. scala中有一个根类型Any，它是所有类的父类
   2. **Null**是所有AnyRef类型的子类，它只有一个值null
   3. **Nothing**是所有Any类型的子类，在开发中通常可以将Nothing类型的值返回非任意变量或者函数，在抛出异常中使用的多
   4. **Unit**表示无值【等价于java的void】,
   5. 它只有一个实例()
   6. 在scala中**低精度**的值向**高精度**的值自动转换（implicit conversion）隐式转换
3. 在scala中，如果一个方法没有形参，则可以省略()

## 2、整型【默认是int】

1. ![1575083035749](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575083035749.png)
2. -0以补码的方式借用给-2147483648，所有int的数值范围是-2147483648到+2147483647

1. 浮点类型【默认是double】
2. 字符类型（char）
   1. 当我们输出一个char类型时，scala会输出该数字对应的字符（码值表 unicode）【Unicode码值表包含ascii】
   2. char 也可以当做数值运算
   3. var num: Char = 'a'+1正确吗？【不正确，因为'a'+1运算的结果是int， int不能隐式转换为char】
   4. ![1575085670386](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575085670386.png)
   5. 存储：字符-》码值-》二进制-》存储
   6. 读取：二进制-》码值-》字符-》读取

## 3、布尔类型

1. **只能是true和false**

## 4、Unit\Null\Nothing

1. Unit：等价于java的void,只有一个值：()
2. Null：只有一个值null，，可以赋给任意的AnyRef，但是不能赋值给AnyVal
3. Nothing：是所有类型的子类，主要用于异常的抛出

## 5、值类型转换

1. 有多种类型的数值混合运算时，系统首先自动将所有数据转换成**容量最大**的那种数据类型，然后再进行计算
2. （byte/short）与char之间不会自动的转换类型
3. （byte/short）与char三者可以计算,计算结果会转成int

## 6、强制类型转换

1. 3.5.toInt==》3【没有四舍五入，直接抹掉小数】
2. 容量大的转成容量小的数据类型

## 7、值类型和String类型的转换

1. 在scala中，**“12.5"不能转成Int**

### 

![1575168634914](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575168634914.png)



# 4、算术运算

# 5、流程控制

**import scala.io._**：下划线表示引入scala.io下所有的子类



## 1、scala没有三目运算，使用if else替代

## 2、scala在控制台输出的方式：StdIn.readLine()

## 3、scala中任意表达式都是有返回值的，即ifelse表达式其实也是有返回结果的，具体返回结果的值取决于满足条件的代码体的最后一行内容

```scala
val myAge = 10
    var res = if (myAge > 8) {
      "myAge 小于 8"
      "age is ok"
    } else if (myAge < 5){
      18
    }
    printf("res=%s",res)
  }
//如果myAge=10=>res=age is ok,如果myAge=3=>res=18,如果myAge=7=>res=()【()：是Unit的唯一一个值】
```

## 4、scala中没有switch，而是使用模式匹配

## 5、循环

1. for推导式，for表达式

   ```scala
   //1、to：前闭后闭:1-10
   for (i <- 1 to 10) {
       println(s"i=$i")
   }
   //2、unit：前闭后开:1-9
   for (i <- 1 unit 10) {
       println(s"i=$i")
   }
   ```

2. 循环守卫

   循环保护式（也称条件判断式），保护式为true则进入循环体内部，为false则跳过，类似于**continue**

   ```scala
   //输出结果：3-10
   for (i <- 1 to 10 if i > 2) {
       println(s"i=$i")
   }
   ```

3. 引入变量

   ```scala
   for (i <- 1 to 10; j = i + 2) {
       println(s"i=$i;j=$j")
   }
   ```

4. 嵌套循环

   ```scala
   for (i <- 1 to 10; j <- 101 to 103) {
       println(s"i=$i;j=$j")
   }
   ```

5. 循环返回值

   将遍历过程中处理的结果返回到一个新的Vector集合中，使用 **yield**关键字

   ```scala
   val res = for (i <- 1 to 10) yield i * 2
   println(s"res = $res")
   //输出结果：res = Vector(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
   val res2 = for (i <- 1 to 10) yield {
       if (i % 2 == 0) {
           "偶数"
       } else
       "奇数"
   }
   println(s"res2 = $res2")
   ```

6. 括号的使用

   ```scala
   for {	i <- 1 to 10
           j = i + 2
       } {
       println(s"i=$i;j=$j")
   }
   ```

7. 循环步长

   ```scala
   //方式一：Range
   for (i <- Range(1,10,2)) {
       println(s"i=$i")
   }
   //方式二：使用循环守卫
   for (i <- 1 to 10 if i % 2 == 0) {
       println(s"i=$i")
   }
   ```

8. while

   推荐使用for：原因：函数或者for循环内部不建议使用外部变量，可通过递归实现

9. do...while

10. for去掉了break和continue关键字

    1. break()

       1. import util.control.Breaks._：使用break()
       2. 在scala中使用函数式的break()来中断：def break(): Nothing = { throw breakException }

    2. breakable()

       breakable()是一个高阶函数【可以接收函数的函数】

       1. ```scala
          def breakable(op: => Unit): Unit = {
              try {
                op
              } catch {
                case ex: BreakControl =>
                  if (ex ne breakException) throw ex
              }
            }
          //op: => Unit 表示接收的参数是一个没有输入，也没有返回值的函数【可以简单理解为一段代码块】
          ```

       2. breakable()对break()抛出的异常做了try-catch

       3. ```scala
          //breakable的()可以使用{}
          breakable {
              for (i <- Range(1, 10, 2)) {
                  println(s"i=$i")
                  break()
              }
          }
          ```

    3. continue

       1. 方式一：for循环体内使用if条件
       2. 方式二：循环守护

# 6、函数式编程

![1575790017944](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575790017944.png)

## 1、简介

1. ![1575777623517](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575777623517.png)

2. ```scala
   object Method2Function {
     def main(args: Array[String]): Unit = {
       val dog = new Dog
       println(dog.sum(2, 3))
   	//函数
       val f1 = dog.sum _
       //函数
       val f2 = (n1: Int, n2: Int) => {
         n1 + n2
       }
   
       println(s"f1=$f1,f1运行=" + f1(3, 4))
       println(s"f2=$f2,f2运行=" + f2(4, 5))
     }
   }
   
   class Dog {
     //方法
     def sum(n1: Int, n2: Int): Int = {
       n1 + n2
     }
   }
   ```

   

## 2、基本语法

![1575789912267](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575789912267.png)

## 3、递归【scala课程-56】

![1575791849839](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1575791849839.png)













