## Gradle
---
Gradle和Maven一样，是一个构建项目的框架。核心是各种各样的Plugin。包括构建Java项目的Plugin,还有War等。与Maven不同，Gradle不提供内建的项目生命周期管理，只是Java plugin向Project中添加了许多的task。这些task依次执行，为我们营造出生命周期一般。

Gradle主要是project 和task。project为容器，task为执行的过程。所有的plugin都是向project中添加配置的property或者添加task。

### Project & Task
gradle 的project 本质上是含有多个task的容器。我们创建的task 和 project中定义的task 都会存在于Project的TaskContainer中。

- 调用Project的task()方法创建task。

```
task hello<<{
  println 'hello'
}
//   <<表示追加的意思，向hello中加入执行过程，等同于doLast

task hello {
  doLast{
    println 'hello'
  }
}
如果要在task的最前面加入执行过程，我们可以用doFirst

task hello{
  doFirst{
    println 'hello'
  }
}

```
（语法规范 应符合groovy）上面的task关键字实际上是一个方法调用，该方法属于Project。Project中存在多个重载的task（）方法。在调用groovy方法时，不用放参数在括号里。

- 通过TaskContainer 的create（）方法来创建task

```
tasks.create(name:'hello')<<{
  println 'hello'
}

```

- 声明task之间的依赖关系

```
task hello(dependsOn:helloFather) << {
  println 'hello'
}
或者
task hello <<　　{
  println 'hello'
}
hello.dependsOn helloFather
```
- 配置task:  一个task 除了执行操作之外，还包括多个property,其中gradle 为每个Task默认定义的Property比如description，logger等。另外每个特定的Task类型还可以包含特定的Property,比如 copy的from和to等。

```
首先在定义Task的时候对Property进行配置
task hello << {
  description = "hello world"
  println description
}

通过闭包配置已有的task
task hello << {
  println description
}
hello{
  description='hello world'
}
或者
hello.description = 'hello world'

还可以通过Task的configure()方法设置
task hello <<{
  println description
}
hello.configure{
  description = 'hello world'
}

```
**注意:Gradle在执行task的时候分为两个阶段，首先配置，然后执行。**

### Gradle 语法
Gradle是一种声明式的构建工具。在执行的时候分两个阶段，先配置再执行。在配置阶段，Gradle将读取所有build.gradle 文件的内容来配置task和Project.(Project和task的property,task之间的依赖关系)

（参考代码看上一章节的。）上面三个Task的完成的功能相同，都是先设置description属性，然后输出到命令行。

事实上，对于每个Task，gradle 都会在Project中创建一个同名的Property,所以我们可以将该Task当做Property来访问。(第二种配置方法的第二种方式)。Gradle还会创建一个同名的方法，该方法接收闭包（第二种配置的第一种方式）。

要了解Gradle，首先要了解Groovy语言中的两个概念。Bean 和闭包的delegate机制。
  - Bean Groovy中的Bean和java中的有个很大的不同。Groovy会为每个字段自动生成getter和setter，并可以通过访问字段来调用getter和setter。
  ```
  class GroovyBean{
    private String name
  }

   def bean = new GroovyBean()
   bean.name = 'GroovyBean is name'
   println bean.name

  ```
  表面上我们直接访问了私有成员变量。实际底层调用了getter和setter方法。Groovy采用这种方法的目的是增加代码的可读性。

  - Groovy 的闭包delegate机制，简单来说就是delegate机制可以使我们将一个闭包中执行代码的作用对象设置成任意对象。
   - Groovy闭包关键字 **this** :跟Java一样指的是定义闭包的封装类。
   - Groovy闭包关键字 **owner** :封装对象(this或者环绕闭包)
   - Groovy闭包关键字 **delegate** :默认情况下和owner一样。但是可以改变。

```
  class Child {
   private String name
}

class Parent {
   Child child = new Child();

   void configChild(Closure c) {
      c.delegate = child
      c.setResolveStrategy Closure.DELEGATE_FIRST
      c()
   }
}

def parent = new Parent()
parent.configChild {
name = "child name"
}

println parent.child.name

```
以上例子中，当我们调用configChild()方法时，我们并没有指定name属性是属于child的，但实际上就是在设置Child的name属性。 默认情况下name被认为是属于Parent的，但是我们在configChild这个方法中做了一下操作，使其不再访问Parent的name属性，而是Child的name。首先在方法中设置delegate接收child。然后将该闭包的ResolveStrategy设置成DELEGATE_FIRST。所以调用configChild方法时闭包代码会被代理到child上。即 这些代码在child上执行。

闭包默认的ResolveStrategy是OWNER_FIRST，即先查找闭包的owner,如果owner存在，则在owner中执行闭包代码。


Groovy 关键字 **as、println、break、case、catch、class、const、continue、def、default、do、else、enum、extends、false、finally、for、goto、if、implements、import、in、instanceof、interface、new、null、package、return、super、switch、this、throw、throws、trait、true、try、while** 我们编码的时候要注意。

#### Groovy闭包语法：
- 定义一个闭包

```

{ [closureParameters -> ] statements }

//[closureparameters -> ]是可选的逗号分隔的参数列表，参数类似于方法的参数列表，这些参数可以是类型化或非类型化的。


//最基本的闭包
{ item++ }                                          
//使用->将参数与代码分离
{ -> item++ }                                       
//使用隐含参数it（后面有介绍）
{ println it }                                      
//使用明确的参数it替代
{ it -> println it }                                
//使用显示的名为参数
{ name -> println name }                            
//接受两个参数的闭包
{ String x, int y ->                                
    println "hey ${x} the value is ${y}"
}
//包含一个参数多个语句的闭包
{ reader ->                                         
    def line = reader.readLine()
    line.trim()
}

```

- 闭包对象 一个闭包其实就是一个groovy.lang.Closure类型的实例


```

//定义一个Closure类型的闭包
def listener = { e -> println "Clicked on $e.source" }      
println listener instanceof Closure
//定义直接指定为Closure类型的闭包
Closure callback = { println 'Done!' }                      
Closure<Boolean> isTextFile = {
    File it -> it.name.endsWith('.txt')                     
}

```

- 调运闭包：其实闭包和C语言的函数指针非常像，我们定义好闭包后调用的方法有如下两种形式：

 - 闭包对象.call(参数)

 - 闭包对象(参数)


 ```
def code = { 123 }
println code() == 123
println code.call() == 123

def isOdd = { int i-> i%2 == 1 }                            
println isOdd(3) == true                                     
println isOdd.call(2) == false

 ```


 **特别注意，如果闭包没定义参数则默认隐含一个名为it的参数**

#### Groovy 闭包参数：

- 普通参数：一个闭包的普通参数定义必须遵循如下一些原则：
        参数类型可选
        参数名字
        可选的参数默认值
        参数必须用逗号分隔


```

def closureWithOneArg = { str -> str.toUpperCase() }
println closureWithOneArg('groovy') == 'GROOVY'

def closureWithOneArgAndExplicitType = { String str -> str.toUpperCase() }
println closureWithOneArgAndExplicitType('groovy') == 'GROOVY'

def closureWithTwoArgs = { a,b -> a+b }
println closureWithTwoArgs(1,2) == 3

def closureWithTwoArgsAndExplicitTypes = { int a, int b -> a+b }
println closureWithTwoArgsAndExplicitTypes(1,2) == 3

def closureWithTwoArgsAndOptionalTypes = { a, int b -> a+b }
println closureWithTwoArgsAndOptionalTypes(1,2) == 3

def closureWithTwoArgAndDefaultValue = { int a, int b=2 -> a+b }
println closureWithTwoArgAndDefaultValue(1) == 3

```
- 隐含参数  当一个闭包没有显式定义一个参数列表时，闭包总是有一个隐式的it参数

```
def greeting = { "Hello, $it!" }
println greeting('World') == 'Hello, World!'

def greeting = { it->"Hello, $it!" }
println greeting('World') == 'Hello, World!'
```

如果你想声明一个不接受任何参数的闭包，必须声明一个空的参数列表：

```
def nullArgMethod = { -> 250}

```

- 可变参数：支持最后一个参数为不定长可变参数。

```
def method1 = {String... args -> args.join('')}
println method1('abc','def')=='abcdef'

def method2 = {String[] args -> args.join('')}
println  method2('abc','def')=='abcdef'

def multiArgsMethod = { int n,String... args ->args.join('')*n
println multiArgsMethod(3,'ab','cd')==abcdabcdabcd

}

```

### Groovy闭包省略调用

```

def debugClosure(int num, String str, Closure closure){  
      //dosomething  
}  

debugClosure(1, "groovy", {  
   println"hello groovy!"  
})

```

可以看见，当闭包作为闭包或方法的最后一个参数时我们可以将闭包从参数圆括号中提取出来接在最后，如果闭包是唯一的一个参数，则闭包或方法参数所在的圆括号也可以省略；对于有多个闭包参数的，只要是在参数声明最后的，均可以按上述方式省略。


### Groovy I/O操作

- 读文件：

```
//把读到的文件行内容全部存入List列表中
def list = new File(baseDir, 'test.txt').collect {it}
//把读到的文件行内容全部存入String数组列表中
def array = new File(baseDir, 'test.txt') as String[]
//把读到的文件内容全部转存为byte数组
byte[] contents = file.bytes

//把读到的文件转为InputStream，切记此方式需要手动关闭流
def is = new File(baseDir,'test.txt').newInputStream()
// do something ...
is.close()

//把读到的文件以InputStream闭包操作，此方式不需要手动关闭流
new File(baseDir,'test.txt').withInputStream { stream ->
    // do something ...
}


```

- 写文件

```

//向一个文件以utf-8编码写三行文字
new File(baseDir,'test.txt').withWriter('utf-8') { writer ->
    writer.writeLine 'Into the ancient pond'
    writer.writeLine 'A frog jumps'
    writer.writeLine 'Water’s sound!'
}
//上面的写法可以直接替换为此写法
new File(baseDir,'test.txt') << '''Into the ancient pond
A frog jumps
Water’s sound!'''
//直接以byte数组形式写入文件
file.bytes = [66,22,11]
//类似上面读操作，可以使用OutputStream进行输出流操作，记得手动关闭
def os = new File(baseDir,'data.bin').newOutputStream()
// do something ...
os.close()
//类似上面读操作，可以使用OutputStream闭包进行输出流操作，不用手动关闭
new File(baseDir,'data.bin').withOutputStream { stream ->
    // do something ...
}

```

- 文件树操作


```
//遍历所有指定路径下文件名打印
dir.eachFile { file ->                      
    println file.name
}
//遍历所有指定路径下符合正则匹配的文件名打印
dir.eachFileMatch(~/.*\.txt/) { file ->     
    println file.name
}
//深度遍历打印名字
dir.eachFileRecurse { file ->                      
    println file.name
}
//深度遍历打印名字，只包含文件类型
dir.eachFileRecurse(FileType.FILES) { file ->      
    println file.name
}
//允许设置特殊标记规则的遍历操作
dir.traverse { file ->
    if (file.directory && file.name=='bin') {
        FileVisitResult.TERMINATE                   
    } else {
        println file.name
        FileVisitResult.CONTINUE                    
    }
}
```

### gradle 增量式构建

如果我们将Gradle的task看做一个黑盒子。 那么我们就可以抽象出输入和输出的概念。一个Task输入操作后产生输出。如果多次执行该Task时 输入和输出都是相同的，那么Gradle认为这样的执行是不必要的，是耗时耗资源的。所以引入了增量式构建的概念。

在增量式构建中，Gradle为每个Task定义了输入(inputs)和输出(outputs),如果在执行一个Task时，如果他的输入和输出和前一次执行时没有发生变化。那么，Gradle认为该Task是最新的(UP-TO-DATE)。因此Gradle将不再执行该Task。  
一个Task的inputs和outputs 可以是一个或者多个文件、可以是文件夹或者是Project的某个Property,甚至是某个闭包所定义的条件。

```
//没有定义inputs 和outputs
task method1{
  def sources =fileTree('sourceDir')

  def description = file('description.txt')

  doLast{
    description.withPrintWriter{ writer ->
      sources.each{ source ->
        writer.println source.txt  
      }
    }
  }
}

修改后
task method1{
  def sources =fileTree('sourceDir')

  def description = file('description.txt')

  inputs.dir sources
  outputs.file description

  doLast{
    description.withPrintWriter{ writer ->
      sources.each{ source ->
        writer.println source.txt  
      }
    }
  }
}

```

相比之下多了两行代码。  
  inputs.dir sources  
  outputs.file description  
当首次执行时 会完整的执行该Task,但是再执行一次的时候命令行会显示

  :method1 UP-TO-DATE

  BUILD SUCCESSFUL

  Total time: 2.938 secs

表示这个Task为最新的了Gradle重复执行。当我们修改了sources 或者description的内容。则Gradle会重新执行该task。

### Gradle 自定义Property

Gradle在默认情况下为Project定义了很多的Property 比较常用的：

  - Project ：Project本身
  - name :Project的名字
  - path ：Project的绝对路径
  - description :Project的描述
  - buildDir :Project构建结果存放路径
  - version :Project的版本号

自定义Property:
    在build.gradle中向Project中添加额外的Property时，我们不能直接定义，而是通过ext来定义：
    例如：
    ext.property1='xxx'
    或者闭包的形式
    ext{
      property1 = 'xxx'
    }
    在定义之后，使用这些Property时我们不需要ext，可以直接访问。

### Gradle 依赖管理

在一个类库横行的时代。免不了要依赖第三方库。Gradle可以配置Repository,配置好以后，Gradle会自动下载这些依赖到本地。
Repository可以是Maven和Ivy的Repository,也可以是本地文件系统。

配置Maven的Repository只要在build.gradle 文件中加入以下代码
      repositories {
       mavenCentral()
      }

Gradle 可以将依赖进行分组。 比如说编译时用这组依赖 运行时使用另一组。需要在自己定义个configuration。在声明依赖时实际上是在设置不同的configuration。
定义configuration可以通过以下方式。

      configurations {
        myDependency
      }
以上我们只定义了一个名为myDependency的configuration ,并没有向其中添加任何依赖。我们可以通过dependencies()方法向myDependency中添加实际的依赖

      dependencies {
       myDependency 'junit:junit:4.12'
      }
以上我们就把junit 添加到依赖myDependency中了。(Plugin 中会预定义很多的依赖给用户使用)

Gradle 还可以填加其他的Project或者文件系统例如
    dependencies {
       compile project(':ProjectB')

      compile files('spring-core.jar', 'spring-aap.jar')
      compile fileTree(include: ['*.jar'], dir: 'libs')
    }
