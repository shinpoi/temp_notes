# Java Remote Debug

简单的使用笔记，详细参照
> https://stackify.com/java-remote-debugging/

环境是OpenJDK8，其他版本也没问题，只是启动debug模式的参数可能略有变化。

### 概述:
简单来说，运行jvm的debug有远程模式，  
在远程端启动jvm后，可以在本地通过CLI工具，指定类、方法、行数来给正在运行的程序添加断点，也可以结合IDE操作。

两个后面会出现的术语, `JPDA`和`JDWP`:
```
Java Platform Debugging Architecture (JPDA) is an extensible set of APIs, part of which is a special debugging protocol called JDWP (Java Debug Wire Protocol).
JDWP is a protocol for communication between the application and the debugger processes, which can be used to troubleshoot a running Java application remotely.
To configure the remote application for debugging, you have to enable the debug mode and specify the parameters for this protocol.
```
> https://stackify.com/java-remote-debugging/

## QuickStart

### remote debug参数
```
java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n ${YouClass}
```
- transport有`dt_socket`和`dt_shmem`两个模式，`dt_shmem`是本地共享内存，IDE的debug大多用的这个。


### 1. CLI示例

#### 1.1 测试用类
- 从键盘拿到输入，然后原样输出
```java
// cat Foo.java

import java.util.Scanner;

public class Foo {

  public static void main(String[] args) {
    Scanner keyboard = new Scanner(System.in);
    while (true) {
      System.out.print("input> ");
      String line = keyboard.nextLine().trim();

      if ("".equals(line)) {  /* line 11 */
        continue;
      }

      myPrint("Your input: " + line);
    }
  }

  private static void myPrint(String text) {
    System.out.println(text);
  }
}
```

#### 1.2 编译 & 执行
```bash
# Terminal A

javac Foo.java
java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n Foo
```

启动后会看到如下输出
```bash
# Terminal A

Listening for transport dt_socket at address: 8000
```

#### 1.3 连接
```bash
# Terminal B

jdb -attach localhost:8000 -sourcepath ./
```
- `sourcepath`指定`Foo.java`的位置. 不指定（或者写错）也能正常工作，只是输出时看不到前后代码


看到这样的输出就表示连接成功了
```bash
# Terminal B

Set uncaught java.lang.Throwable
Set deferred uncaught java.lang.Throwable
Initializing jdb ...
>
```

指定断点（通过方法）
```bash
# Terminal B

> stop in Foo.myPrint(java.lang.String)
Set breakpoint Foo.myPrint(java.lang.String)
```

指定断点（通过行数）
```bash
# Terminal B

> stop in Foo:11
Set breakpoint Foo:11
```

在jvm那边随便敲点东西回车
```bash
# Terminal A

input> Hello
```
这里本来应该有`"Your input: Hello"`输出的，可以看到input后停住了

回到jdb这边，能看到这样的输出
```bash
# Terminal B

> stop in Foo:11
Set breakpoint Foo:11
>
Breakpoint hit: "thread=main", Foo.main(), line=11 bci=27
11          if ("".equals(line)) {

main[1]
```

mian后面可以进行其它操作，比如`list`(查看上下代码), `step`（步进）, `step up`(到下个断点), `locals`(显示所有局部变量)等


### 1.4 扩展
---
#### 1.4.1 Connectors
例子里只用了最基本的Socket Attaching Connector，dt_socket模式下默认用打开的。  
可以通过定制Connectors实现更复杂的操作。
```
In this simple example, you just use a Socket Attaching Connector, 
which is enabled by default when the dt_socket transport is configured and the VM is running in the server debugging mode.
```
> https://stackify.com/java-remote-debugging/

#### 1.4.2 编译参数
上面的例子说了locals可以打印所有局部变量，但实际照做了的话会有如下输出:
```bash
# Terminal A

main[1] locals
Local variable information not available.  Compile with -g to generate variable information
```

理由是javac编译时默认不生成变量的debug信息
```bash
$ javac -help
Usage: javac <options> <source files>
where possible options include:
  -g                         Generate all debugging info
  -g:none                    Generate no debugging info
  -g:{lines,vars,source}     Generate only some debugging info
  ...
```

oracle的文档里更详细: https://docs.oracle.com/javase/8/docs/technotes/tools/windows/javac.html
```
-g
Generates all debugging information, including local variables. 
By default, only line number and source file information is generated.

-g:none
Does not generate any debugging information.

-g:[keyword list]
Generates only some kinds of debugging information, 
specified by a comma separated list of keywords. Valid keywords are:
    source: Source file debugging information.
    lines: Line number debugging information.
    vars: Local variable debugging information.
```

所以再来一次：
```bash
# Terminal A

javac -g Foo.java
java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n Foo

Listening for transport dt_socket at address: 8000
input> Hello
```

```bash
# Terminal B

Set uncaught java.lang.Throwable
Set deferred uncaught java.lang.Throwable
Initializing jdb ...
> stop in Foo:11
Set breakpoint Foo:11
>
Breakpoint hit: "thread=main", Foo.main(), line=11 bci=27
11          if ("".equals(line)) {

main[1] locals
Method arguments:
args = instance of java.lang.String[0] (id=607)
Local variables:
keyboard = instance of java.util.Scanner(id=608)
line = "Hello"
main[1]
```

---

## 2. Intellij
实际调试时通过CLI太墨迹了，万能的IDE当然也提供了remote debug功能
- 官方: https://www.jetbrains.com/help/idea/run-debug-configuration-remote-debug.html
- 俺更喜欢图文并茂的这篇: https://docs.alfresco.com/5.2/tasks/sdk-debug-intellij.html

简单来说:
1. 设置`run configuration`, 在左边的`Remote`栏里新建一个，设置host和端口。  
（设置好后Intellj会生成远程端需要的java的运行参数，贴心w）
2. 切换到刚才的设置，会发现不能点执行，只有debug按钮可以按
3. 和一般的debug一样，在代码里打好断点
4. 远程端执行程序
5. IDE里点debug，没报错就是连接上了
6. 随便做点啥触发代码跑到断点的位置，IDE里就会像本地debug一样跳到断点并显示变量信息。


注意一点，mvn/gradle请自行添加编译参数，这俩默认是不加`-g`的  
直接拷class文件的话，请先设置`run configuration`，然后再编译，然后再拷贝class。不然编译出的class文件也是不带debug信息的。

以上。
