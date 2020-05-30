### Java设计模式

#### 1  设计模式的重要性

软件工程中， 设计模式（design pattern）是对 软件设计中普遍存在（反复出现）的各种问题，所提出的 解决方案。这个术语是由埃里希·伽玛（Erich Gamma）等人在1990年代从建筑设计领域引入到计算机科学的。

**设计模式的目的**

编写软件过程中，程序员面临着来自 耦合性，内聚性以及可维护性，可扩展性，重用性，灵活性 等多方面的挑战，设计模式是为了让 程序( 软件) ，具有更好的：

- 代码重用性 (即：相同功能的代码，不用多次编写)
- 可读性 (即：编程规范性, 便于其他程序员的阅读和理解)
- 可扩展性 (即：当需要增加新的功能时，非常的方便，称为可维护）
- 可靠性 (即：当我们增加新的功能后，对原来的功能没有影响)
-  使程序呈现 高内聚， 低耦 合的特性

##### 1.1 七大原则

- 单一职责原则
- 接口隔离原则
- 依赖倒转原则
- 里式替换原则
- 开闭原则
- 迪米特原则
- 合成复用原则

###### 1.1.1 单一职责原则

**基本介绍：**

对类来说的，即一个类应该只负责一项职责。如类A负责两个不同职责：职责1，职责2。当职责1需求变更而改变A时，可能造成职责2执行错误，所以需要将类A的粒度分解为A1，A2

​	单 一职责原则注意事项

- 降低类的复杂性，一个类只维护一项职责。
- 提高类的可读性，可维护性
- 降低变更引起的风险
- 通常情况下，我们应当遵守单一职责原则，只有逻辑足够简单，才能在代码级别违反单一职责原则；只有类中方法足够少，可以再方法级别保持单一职责原则。

###### 1.1.2 接口隔离原则

客户端不应该依赖它不需要的接口，即一个类对另一个类的依赖应该建立在最小的接口上。

###### 1.1.3 依赖倒转原则

- 高层模块不应该依赖底层模块，二者都应该依赖其抽象
- 抽象不应该依赖细节，细节应该依赖抽象
- 依赖倒转的核心思想是面向接口编程
- 依赖倒转原则是基于这样的设计理念：相对于细节的多变性，抽象的东西要稳定的多，以抽象为基础搭建的框架比以细节为基础搭建的框架要稳定的多，在java中，抽象是指的接口或抽象类，细节就是具体的实现类。
- 使用接口或抽象类的目的是指定好规范，而不涉及具体的操作，把展现细节的东西交给他们的实现类去完成。

```java
public class DependecyInversion {

   public static void main(String[] args) {
      Person person = new Person();
      person.receive(new Email());
   }

}


class Email {
   public String getInfo() {
      return "电子邮件信息: hello,world";
   }
}

//完成Person接收消息的功能
//方式1分析
//1. 简单，比较容易想到
//2. 如果我们获取的对象是 微信，短信等等，则新增类，同时Perons也要增加相应的接收方法
//3. 解决思路：引入一个抽象的接口IReceiver, 表示接收者, 这样Person类与接口IReceiver发生依赖
//   因为Email, WeiXin 等等属于接收的范围，他们各自实现IReceiver 接口就ok, 这样我们就符号依赖倒转原则
class Person {
   public void receive(Email email ) {
      System.out.println(email.getInfo());
   }
}
```

使用依赖倒转原则去改进下代码

```java
public class DependecyInversion {

   public static void main(String[] args) {
      //客户端无需改变
      Person person = new Person();
      person.receive(new Email());
      
      person.receive(new WeiXin());
   }

}

//定义接口
interface IReceiver {
   public String getInfo();
}

class Email implements IReceiver {
   public String getInfo() {
      return "电子邮件信息: hello,world";
   }
}

//增加微信
class WeiXin implements IReceiver {
   public String getInfo() {
      return "微信信息: hello,ok";
   }
}

//方式2
class Person {
   //这里我们是对接口的依赖
   public void receive(IReceiver receiver ) {
      System.out.println(receiver.getInfo());
   }
}
```

依赖关系的三种传递方式

- 通过接口
- 构造方法传递
- 通过set传递

依赖倒转原则需要注意的事项：

- 底层模块尽量都要有接口类或接口，或者两者都有，	程序稳定性更好。
- 变量的声明类型尽量都是接口或抽象类，这样我们的变量引用和实际对象间，就存在一个缓冲层，利于程序扩展和优化
- 继承时遵循里氏替换原则

###### 1.1.4 里氏替换原则

OO中继承的思考和说明

- 继承包含这样一种含义：父类中凡是已经实现好的方法，实际上是在设定规范和契约，虽然他不强制所有子类都需要遵守这些契约，但是如果子类对这些已经实现的方法任意修改，就会对整个继承体系造成破坏。
- 继承在给程序设计带来便利的同时也带来了弊端，比如使用继承会给程序带来侵入性，程序的可移植性降低，增加对象间的耦合性，如果一个类被其他类所继承，则当这个类需要修改时，必须考虑所有子类，并且父类修改后，所有涉及到子类的动能都有可能发生故障。
- 提出问题，如何正确使用继承？  **里式替换原则**

**基本介绍**

- 里氏替换原则(Liskov Substitution Principle)在 1988 年，由麻省理工学院的以为姓里的女士提出的。
-  如果对每个类型为 T1 的对象 o1，都有类型为 T2 的对象 o2，使得以 T1 定义的所有程序 P 在所有的对象 o1 都
  代换成 o2 时，程序 P 的行为没有发生变化，那么类型 T2 是类型 T1 的子类型。 换句话说，所有引用基类的地
  方必须能透明地使用其子类的对象。
- 在使用继承时，遵循里氏替换原则，在 子类中尽量不要重写父类的方法
- 里氏替换原则告诉我们，继承实际上让两个类耦合性增强了，在适当的情况下，可以通过聚合、组合、依赖来
  解决问题。

问题

```java
public class Liskov {

   public static void main(String[] args) {
      // TODO Auto-generated method stub
      A a = new A();
      System.out.println("11-3=" + a.func1(11, 3));
      System.out.println("1-8=" + a.func1(1, 8));

      System.out.println("-----------------------");
      B b = new B();
      System.out.println("11-3=" + b.func1(11, 3));//这里本意是求出11-3
      System.out.println("1-8=" + b.func1(1, 8));// 1-8
      System.out.println("11+3+9=" + b.func2(11, 3));
      
      

   }

}

// A类
class A {
   // 返回两个数的差
   public int func1(int num1, int num2) {
      return num1 - num2;
   }
}

// B类继承了A
// 增加了一个新功能：完成两个数相加,然后和9求和
class B extends A {
   //这里，重写了A类的方法, 可能是无意识
   public int func1(int a, int b) {
      return a + b;
   }

   public int func2(int a, int b) {
      return func1(a, b) + 9;
   }
}
```

**问题：**我们发现原来运行正常的相减功能发生了错误。原因就是类 B 无意中重写了父类的方法，造成原有功能出现错误。在实际编程中，我们常常会通过重写父类的方法完成新的功能，这样写起来虽然简单，但整个继承体系的复用性会比较差。特别是运行多态比较频繁的时候

**通用的做法是：** 原来的父类和子类都继承一个更通俗的基类，原有的继承关系去掉，采用 依赖 ， 聚合 ， 组合等
关系代替.

```java
public class Liskov {

   public static void main(String[] args) {
      // TODO Auto-generated method stub
      A a = new A();
      System.out.println("11-3=" + a.func1(11, 3));
      System.out.println("1-8=" + a.func1(1, 8));

      System.out.println("-----------------------");
      B b = new B();
      //因为B类不再继承A类，因此调用者，不会再func1是求减法
      //调用完成的功能就会很明确
      System.out.println("11+3=" + b.func1(11, 3));//这里本意是求出11+3
      System.out.println("1+8=" + b.func1(1, 8));// 1+8
      System.out.println("11+3+9=" + b.func2(11, 3));
      
      
      //使用组合仍然可以使用到A类相关方法
      System.out.println("11-3=" + b.func3(11, 3));// 这里本意是求出11-3
      

   }

}

//创建一个更加基础的基类
class Base {
   //把更加基础的方法和成员写到Base类
}

// A类
class A extends Base {
   // 返回两个数的差
   public int func1(int num1, int num2) {
      return num1 - num2;
   }
}

// B类继承了A
// 增加了一个新功能：完成两个数相加,然后和9求和
class B extends Base {
   //如果B需要使用A类的方法,使用组合关系
   private A a = new A();
   
   //这里，重写了A类的方法, 可能是无意识
   public int func1(int a, int b) {
      return a + b;
   }

   public int func2(int a, int b) {
      return func1(a, b) + 9;
   }
   
   //我们仍然想使用A的方法
   public int func3(int a, int b) {
      return this.a.func1(a, b);
   }
}
```

###### 1.1.5 开闭原则

- 开闭原则（Open Closed Principle）是编程中 最基础、最重要的设计原则
- 一个软件实体如类，模块和函数应该 对扩展开放( 对提供方)，对 修改关闭( 对使用方)。用抽象构建框架，用实
  现扩展细节。
- 当软件需要变化时，尽量 通过扩展软件实体的行为来实现变化，而不是 通过修改已有的代码来实现变化。
- 编程中遵循其它原则，以及使用设计模式的目的就是遵循开闭原则。

```java
public class Ocp {

   public static void main(String[] args) {
      //使用看看存在的问题
      GraphicEditor graphicEditor = new GraphicEditor();
      graphicEditor.drawShape(new Rectangle());
      graphicEditor.drawShape(new Circle());
      graphicEditor.drawShape(new Triangle());
   }

}

//这是一个用于绘图的类 [使用方]
class GraphicEditor {
   //接收Shape对象，然后根据type，来绘制不同的图形
   public void drawShape(Shape s) {
      if (s.m_type == 1)
         drawRectangle(s);
      else if (s.m_type == 2)
         drawCircle(s);
      else if (s.m_type == 3)
         drawTriangle(s);
   }

   //绘制矩形
   public void drawRectangle(Shape r) {
      System.out.println(" 绘制矩形 ");
   }

   //绘制圆形
   public void drawCircle(Shape r) {
      System.out.println(" 绘制圆形 ");
   }
   
   //绘制三角形
   public void drawTriangle(Shape r) {
      System.out.println(" 绘制三角形 ");
   }
}

//Shape类，基类
class Shape {
   int m_type;
}

class Rectangle extends Shape {
   Rectangle() {
      super.m_type = 1;
   }
}

class Circle extends Shape {
   Circle() {
      super.m_type = 2;
   }
}

//新增画三角形
class Triangle extends Shape {
   Triangle() {
      super.m_type = 3;
   }
}
```

**优缺点分析**

- 优点是比较好理解，简单易操作。
-  缺点是违反了设计模式的 ocp 原则，即对扩展开放(提供方)，对修改关闭(使用方)。即当我们给类增加新功能的
  时候，尽量不修改代码，或者尽可能少修改代码.
- 比如我们这时要新增加一个图形种类 三角形，我们需要做如下修改，修改的地方较多

**改进的思路分析**

思路：把创建 Shape 类做成抽象类，并提供一个 抽象的draw 方法，让 子类去实现即可，这样我们有新的图形
种类时，只需要让新的图形类继承 Shape，并实现 draw 方法即可，修 使用方的代码就不需要修 -> 满足了开闭原则

```java
public class Ocp {

   public static void main(String[] args) {
      //使用看看存在的问题
      GraphicEditor graphicEditor = new GraphicEditor();
      graphicEditor.drawShape(new Rectangle());
      graphicEditor.drawShape(new Circle());
      graphicEditor.drawShape(new Triangle());
      graphicEditor.drawShape(new OtherGraphic());
   }

}

//这是一个用于绘图的类 [使用方]
class GraphicEditor {
   //接收Shape对象，调用draw方法
   public void drawShape(Shape s) {
      s.draw();
   }

   
}

//Shape类，基类
abstract class Shape {
   int m_type;
   
   public abstract void draw();//抽象方法
}

class Rectangle extends Shape {
   Rectangle() {
      super.m_type = 1;
   }

   @Override
   public void draw() {
      // TODO Auto-generated method stub
      System.out.println(" 绘制矩形 ");
   }
}

class Circle extends Shape {
   Circle() {
      super.m_type = 2;
   }
   @Override
   public void draw() {
      // TODO Auto-generated method stub
      System.out.println(" 绘制圆形 ");
   }
}

//新增画三角形
class Triangle extends Shape {
   Triangle() {
      super.m_type = 3;
   }
   @Override
   public void draw() {
      // TODO Auto-generated method stub
      System.out.println(" 绘制三角形 ");
   }
}

//新增一个图形
class OtherGraphic extends Shape {
   OtherGraphic() {
      super.m_type = 4;
   }

   @Override
   public void draw() {
      // TODO Auto-generated method stub
      System.out.println(" 绘制其它图形 ");
   }
}
```

###### 1.1.6 迪米特法则

- 一个对象应该对其他对象保持最少的了解
- 类与类关系越密切，耦合度越大
-  迪米特法则(Demeter Principle)又叫 最少知道原则，即一个类 对自己依赖的类知道的越少越好。也就是说，对于被依赖的类不管多么复杂，都尽量将逻辑封装在类的内部。对外除了提供的 public 方法，不对外泄露任何信息
-  迪米特法则还有个更简单的定义：只与直接的朋友通信
-  直接的朋友：每个对象都会与其他对象有 耦合关系，只要两个对象之间有耦合关系，我们就说这两个对象之间
  是朋友关系。耦合的方式很多，依赖，关联，组合，聚合等。其中，我们称出现 **成员变量**， **方法参数**， **方法返**
  **回值中的类**为直接的朋友，而出现在 局部变量中的类不是直接的朋友。也就是说，陌生的类最好不要以局部变
  量的形式出现在类的内部

```java
public class Demeter1 {

   public static void main(String[] args) {
      //创建了一个 SchoolManager 对象
      SchoolManager schoolManager = new SchoolManager();
      //输出学院的员工id 和  学校总部的员工信息
      schoolManager.printAllEmployee(new CollegeManager());

   }

}


//学校总部员工类
class Employee {
   private String id;

   public void setId(String id) {
      this.id = id;
   }

   public String getId() {
      return id;
   }
}


//学院的员工类
class CollegeEmployee {
   private String id;

   public void setId(String id) {
      this.id = id;
   }

   public String getId() {
      return id;
   }
}


//管理学院员工的管理类
class CollegeManager {
   //返回学院的所有员工
   public List<CollegeEmployee> getAllEmployee() {
      List<CollegeEmployee> list = new ArrayList<CollegeEmployee>();
      for (int i = 0; i < 10; i++) { //这里我们增加了10个员工到 list
         CollegeEmployee emp = new CollegeEmployee();
         emp.setId("学院员工id= " + i);
         list.add(emp);
      }
      return list;
   }
}

//学校管理类

//分析 SchoolManager 类的直接朋友类有哪些 Employee、CollegeManager
//CollegeEmployee 不是 直接朋友 而是一个陌生类，这样违背了 迪米特法则 
class SchoolManager {
   //返回学校总部的员工
   public List<Employee> getAllEmployee() {
      List<Employee> list = new ArrayList<Employee>();
      
      for (int i = 0; i < 5; i++) { //这里我们增加了5个员工到 list
         Employee emp = new Employee();
         emp.setId("学校总部员工id= " + i);
         list.add(emp);
      }
      return list;
   }

   //该方法完成输出学校总部和学院员工信息(id)
   void printAllEmployee(CollegeManager sub) {
      
      //分析问题
      //1. 这里的 CollegeEmployee 不是  SchoolManager的直接朋友
      //2. CollegeEmployee 是以局部变量方式出现在 SchoolManager
      //3. 违反了 迪米特法则 
      
      //获取到学院员工
      List<CollegeEmployee> list1 = sub.getAllEmployee();
      System.out.println("------------学院员工------------");
      for (CollegeEmployee e : list1) {
         System.out.println(e.getId());
      }
      //获取到学校总部员工
      List<Employee> list2 = this.getAllEmployee();
      System.out.println("------------学校总部员工------------");
      for (Employee e : list2) {
         System.out.println(e.getId());
      }
   }
}
```

迪米特法则注意事项和原则

- 迪米特法则的核心是降低类之间的耦合
- 由于每个类都降低了不必要的依赖，因此迪米特法则只是要求降低类间的关系，并不是要求完全没有依赖关系

###### 1.1.7 合成复用原则

原则上是尽量使用合成、聚合的方式，而不是使用继承

###### 1.1.8 设计原则核心思想

- 找出应用中需要变化之处，把他们独立出来，不要和那些不需要变化的代码混在一起。
- 针对接口编程，而不是针对实现编程
- 为了交互对象间的松耦合设计而努力

#### 2 设计模式

设计模式介绍：

- 设计模式是程序员在面对同类软件工程设计问题所总结出来的有用的经验， 模式不是代码，而是 某类问题的通
  用解决方案，设计模式（Design pattern）代表了最佳的实践。这些解决方案是众多软件开发人员经过相当长的
  一段时间的试验和错误总结出来的。
- 设计模式的本质提高  软件的维护性，通用性和扩展性，并降低软件的复杂度

设计模式类型：

设计模式分为23种

创建型模式：单例模式、抽象工厂模式、原型模式、建造者模式、工厂方法模式

结构型模式：适配器模式、桥接模式、装饰模式、组合模式、外观模式、享元模式、代理模式

行为型模式：模板方法模式、命令模式、访问者模式、迭代器模式、观察者模式、中介者模式、备忘录模式、解释器模式、状态模式、策略模式、职责链模式

##### 2.1 单例模式

所谓的单例设计模式，就是采取一定的方法保证整个软件系统中，对某个类只能有一个对象实例，并且只提供一个取得该对象实例的方法。

比如Hibernate的SessionFactory，它充当数据存储源的代理，并负责创建Session对象，SessionFactory并不是轻量级的，一般情况下通常只需要一个SessionFactory就够了，这就是使用到了单例模式

单例模式有8种方式：饿汉式（静态常量），饿汉式（静态代码块），懒汉式（线程不安全），懒汉式（线程安全，同步方法），懒汉式（线程安全，同步代码块），双重检查，静态内部类，枚举。

###### 2.1.1 饿汉式(静态变量)

```java
class SingleTon{
    //构造器私有化，外部不能new
    private SingleTon(){

    }
    //本类内部创建对象实例
    private final static SingleTon instance = new SingleTon();
    //提供一个公有的静态方法，返回实例对象
    public static SingleTon getInstance(){
        return instance;
    }
}
```

优缺点说明：

优点：这种写法简单，就是在类加载的时候完成了实例化，避免了线程同步问题

缺点：在类加载的时候就完成了实例化，没有达到Lazy Loading的效果，如果从开始至终从未使用过这个实例，则会造成内存浪费。

​	这种方式基于ClassLoder机制避免了多线程的同步问题，不过，instance在类装载时就实例化，在单例模式中大多数就是调用getInstance方法，但是导致类装载的原因有很多种，因此不能确定有其他方式（或者其他的静态方法）导致的类装载，这时候初始化Instance没有达到LacyLoading的效果。

结论：这种单例模式可用，可能造成资源浪费

###### 2.1.2 饿汉式（静态代码块）

```java
class SingleTon{
    //构造器私有化，外部不能new
    private SingleTon(){

    }
    //本类内部创建对象实例
    private static SingleTon instance;
    static {//在静态代码块中，创建单例对象
        instance = new SingleTon();
    }
    //提供一个公有的静态方法，返回实例对象
    public static SingleTon getInstance(){
        return instance;
    }
}
```

###### 2.1.3懒汉式（线程不安全）

```java
class SingleTon{
    //构造器私有化，外部不能new
    private SingleTon(){

    }
    //本类内部创建对象实例
    private static SingleTon instance;

    //提供一个公有的静态方法，返回实例对象
    public static SingleTon getInstance(){
        if(instance == null){
            instance = new SingleTon();
        }
        return instance;
    }
}
```

- 起到了 Lazy Loading 的效果，但是只能在单线程下使用。
- 如果在多线程下，一个线程进入了 if (singleton == null)判断语句块，还未来得及往下执行，另一个线程也通过
  了这个判断语句，这时便会 产生多个实例。所以在多线程环境下不可使用这种方式
- 结论：在实际开发过程中，不要使用这种模式

###### 2.1.4 懒汉式（线程安全，同步方法）

```java

class SingleTon{
    //构造器私有化，外部不能new
    private SingleTon(){

    }
    //本类内部创建对象实例
    private static SingleTon instance;

    //提供一个公有的静态方法，返回实例对象
    public static synchronized SingleTon getInstance(){
        if(instance == null){
            instance = new SingleTon();
        }
        return instance;
    }
}
```

- 解决了线程安全问题
- 效率太低了，每个线程在想获得类的实例的时候，执行getInstance()方法都要进行同步，而这个方法只执行一次实例化代码就够了，后面的想获得该实例，直接return就行了。方法执行同步效率太低
- 结论：在实际开发中，不推荐使用这种方式

###### 2.1.5 双重检查

```java
class SingleTon{
    //构造器私有化，外部不能new
    private SingleTon(){

    }
    //本类内部创建对象实例
    private static SingleTon instance;

    //提供一个公有的静态方法，返回实例对象
    public static synchronized SingleTon getInstance(){
        if(instance == null){
            synchroniezd(SingleTon.class){
                instance = new SingleTon();
            }
        }
        return instance;
    }
}
```

优缺点说明：

- double-check概念是多线程开发中常使用到的，在代码中所示，我们进行了两次if(instance == null)检查，这样就能保证线程安全了
- 实例代码只用了一次，后面再次访问时，判断if(instance == null)直接return实例化对象，也避免了反复进行方法同步
- 结论：延迟加载，效率较高
- 在实际开发中，推荐使用这这单例模式

###### 2.1.6 静态内部类

```java
class SingleTon{
    private SingleTon(){

    }
    //写一个静态内部类，该类中有个静态属性SingTon
    private static class SingleTonInstance{
        private static final SingleTon INSTANCE = new SingleTon();
    }

    //提供一个静态的公有方法，SingleTonInstance.INSTANCE
    public static synchronized SingleTon getInstance(){
        return SingleTonInstance.INSTANCE;
    }
}
```

- 这种方式采用类的装载机制来保证初始化实例的时候只有一个线程
- 静态内部类方式在Singleton类被装载时并不会立即实例化，而是在需要实例化的时，调用getInstance方法才会装载SingleTonInstance类，从而完成Singleton的实例化
- 类的静态属性只会在第一次加载类的时候初始化，所以在这里，JVM帮助我们保证了线程的安全性，在类进行初始化时，别的线程是无法进入的。
- 优点：避免了线程不安全，利用了静态内部类的特点实现延迟加载，效率高
- 结论：推荐使用

###### 2.1.7 枚举

```java
enum  SingleTon{
    INSTANCE;
}
```

- 这借助 JDK1.5 中添加的枚举来实现单例模式。不仅能避免多线程同步问题，而且还能防止反序列化重新创建
  新的对象
- 这种方式是 Effective Java  作者 Josh Bloch  提倡的方式
- 结论：推荐使用

###### 注意事项

- 单例模式保证了系统内存中只有一个对象，节省了系统资源，对于一些需要频繁创建销毁的对象，使用单例模式可以提高系统性能。
- 当想实例化一个单例模式对象的时候，必须要使用相应的获取对象的放大，而不是使用new
- 单例模式的使用场景：需要频繁进行创建和销毁的对象，创建对象时耗时过多或消耗资源过多（即重量级对象），但又经常用到的对象、工具类对象、频繁访问数据库或文件的对象（比如数据源、session工厂）等。

##### 2.2 工厂模式

###### 2.2.1 简单工厂模式

基本介绍：

简单工厂模式是数据创建型模式，是工厂模式的一种，简单工厂模式是由一个工厂对象决定创建出哪一种产品类的实例，简单工厂模式是工厂模式家族中最简单最实用的模式。

工厂模式设计方案：定义一个创建对象的抽象方法，由子类决定要实例化的类，工厂模式将对象的实例化推迟到子类

###### 2.2.2 工厂方法模式

工厂方法模式：定义了一个**创建对象的抽象方法**，由子类决定要实例化的类，工厂方法模式将对象的实例化推迟到子类。

###### 2.2.3 抽象工厂模式

- 抽象工厂模式：定义了一个interface用于创建相关或有依赖的对象簇，而无需指明具体的类
- 抽象工厂模式可以将简单工厂模式和工厂方法模式进行整合。
- 从设计层面看，抽象工厂模式就是对简单工厂模式的改进（或者成为进一步的抽象）
- 将工厂抽象成两层，AbsFactory和具体实现的工厂子类，程序员可根据创建对象类型使用对应的工厂子类，这样将单个的简单工厂变成了工厂簇，更有利于代码的维护和扩展。

###### 2.3.4 工厂模式小结

意义：将实例化对象的代码提取出来，放到一个类中统一管理和维护，达到和主项目的依赖关系的解耦，从来提高项目的可扩展性和维护性。

设计模式的依赖抽象原则

- 创建对象时，不要直接new类，而是把这个new类的动作放在一个工厂的方法中，并返回。
- 不要让类继承具体类，而是继承抽象类，或者实现interface
- 不要覆盖类中已经实现的方法

##### 2.3 原型模式

传统方式优缺点：

- 优点是比较好理解，简单易操作
- 在创建对象时，总是要重新获取原始对象的属性，如果创建对象比较复杂时，效率较低
- 总是需要重新初始化对象，而不是动态的获取对象运行时的状态，不够灵活

改进思路： Java中Object类是所有类的跟类，Object类提供了一个clone方法，该方法可以将一个Java对象复制一份，但是需要实现clone的Java类要实现Cloneable，该接口表示该类能够复制，且具有复制的能力。

###### 2.3.1 原型模式介绍

原型模式是指，用原型实例指定创建对象的种类，并且通过拷贝这些原型，创建新的对象

原型模式是种创建型模式，运行一个对象再创建另一个可定制对象，无需知道创建的细节

工作原理：通过一个将原型对象传给那个要发动创建的对象，这个要发动创建的对象通过原型对象拷贝他们自己来实施创建，`即对象.clone`

###### 2.3.2 浅拷贝

- 对于数据类型是基本数据类型的成员变量，浅拷贝会直接进行值传递，也就是将该属性值复制一份给新的对象

- 对于数据类型是引用数据类型的成员变量，比如说成员变量是某个数组、某个类的对象等，那么浅拷贝会进行
  引用传递，也就是只是将该成员变量的引用值（内存地址）复制一份给新的对象。因为实际上两个对象的该成
  员变量都指向同一个实例。在这种情况下，在一个对象中修改该成员变量会影响到另一个对象的该成员变量值

- 深拷贝是默认的clone（）方法来实现

###### 2.3.3 深拷贝

- 复制对象的所有基本类型为成员变量值
- 为所有引用数据类型的成员变量申请存储空间，并复制每个引用数据类型成员变量所引用的对象，直到该对象
  可达的所有对象。也就是说， 对象进行深拷贝要对整个对象( 包括对象的引用类型) 进行拷贝

- 深拷贝实现方式 1：重写 clone 方法来实现深拷贝
- 深拷贝实现方式 2：通过 对象序列化实现深拷贝(推荐)

浅拷贝：

```java
//深拷贝 - 方式 1 使用clone 方法
	@Override
	protected Object clone() throws CloneNotSupportedException {
		
		Object deep = null;
		//这里完成对基本数据类型(属性)和String的克隆
		deep = super.clone(); 
		//对引用类型的属性，进行单独处理
		DeepProtoType deepProtoType = (DeepProtoType)deep;
		deepProtoType.deepCloneableTarget  = (DeepCloneableTarget)deepCloneableTarget.clone();
		
		// TODO Auto-generated method stub
		return deepProtoType;
	}
```

浅拷贝：

```java
//深拷贝 - 方式2 通过对象的序列化实现 (推荐)
	
	public Object deepClone() {
		
		//创建流对象
		ByteArrayOutputStream bos = null;
		ObjectOutputStream oos = null;
		ByteArrayInputStream bis = null;
		ObjectInputStream ois = null;
		
		try {
			
			//序列化
			bos = new ByteArrayOutputStream();
			oos = new ObjectOutputStream(bos);
			oos.writeObject(this); //当前这个对象以对象流的方式输出
			
			//反序列化
			bis = new ByteArrayInputStream(bos.toByteArray());
			ois = new ObjectInputStream(bis);
			DeepProtoType copyObj = (DeepProtoType)ois.readObject();
			
			return copyObj;
			
		} catch (Exception e) {
			// TODO: handle exception
			e.printStackTrace();
			return null;
		} finally {
			//关闭流
			try {
				bos.close();
				oos.close();
				bis.close();
				ois.close();
			} catch (Exception e2) {
				// TODO: handle exception
				System.out.println(e2.getMessage());
			}
		}
		
	}
```

2.3.4 原型模式注意事项

- 创建新的对象比较复杂时，可以使用原型模式简化对象的创建过程，同时也可以提高效率
- 不用重新初始化对象，而是动态的获取对象运行时的状态
- 如果原石对象的发生改变时，其他克隆对象也会发生相应的变化，无需修改代码
- 在实现深克隆的时候可能需要比较复杂的代码
- 缺点：需要为每个类配备一个克隆方法，这对于全新的类来说并不是很那，但是对已有的类进行改造，需要修改其源码，违反了OCP原则。需要注意

##### 2.4 建造者模式

- 建造者模式，又叫生成器模式，是一种对象的构建模式，他可以将复杂的对象建造过程抽象出来，使这个抽象过程的不同实现方法可以构造出不同表现的对象

- 建造者模式是一步一步创建一个复杂的对象，它允许用户只通过指定复杂对象的类型和内容就可以构建他们，用户不知道内部的具体实现细节。

建造者模式的四个对象：

- product（产品角色）：一个具体的产品对象
- Builder （抽象建造者）：创建一个product对象的各个内部指令的接口/抽象类
- ConcreteBuilder(具体建造者) ：实现接口，构建和装配各个部件
- Director（指挥者）： 构建一个Builder对象，他主要是用于创建一个复杂的对象，他主要有两个作用，一是：隔离了用户和对象的生产过程。二是：控制产品对象的生产过程

```java
//产品->Product
public class House {
	private String baise;
	private String wall;
	private String roofed;
	public String getBaise() {
		return baise;
	}
	public void setBaise(String baise) {
		this.baise = baise;
	}
	public String getWall() {
		return wall;
	}
	public void setWall(String wall) {
		this.wall = wall;
	}
	public String getRoofed() {
		return roofed;
	}
	public void setRoofed(String roofed) {
		this.roofed = roofed;
	}
```

```java
// 抽象的建造者
public abstract class HouseBuilder {

   protected House house = new House();
   
   //将建造的流程写好, 抽象的方法
   public abstract void buildBasic();
   public abstract void buildWalls();
   public abstract void roofed();
   
   //建造房子好， 将产品(房子) 返回
   public House buildHouse() {
      return house;
   }
```

```java
public class CommonHouse extends HouseBuilder {

   @Override
   public void buildBasic() {
      // TODO Auto-generated method stub
      System.out.println(" 普通房子打地基5米 ");
   }

   @Override
   public void buildWalls() {
      // TODO Auto-generated method stub
      System.out.println(" 普通房子砌墙10cm ");
   }

   @Override
   public void roofed() {
      // TODO Auto-generated method stub
      System.out.println(" 普通房子屋顶 ");
   }
```

```java
//指挥者，这里去指定制作流程，返回产品
public class HouseDirector {
   
   HouseBuilder houseBuilder = null;

   //构造器传入 houseBuilder
   public HouseDirector(HouseBuilder houseBuilder) {
      this.houseBuilder = houseBuilder;
   }

   //通过setter 传入 houseBuilder
   public void setHouseBuilder(HouseBuilder houseBuilder) {
      this.houseBuilder = houseBuilder;
   }
   
   //如何处理建造房子的流程，交给指挥者
   public House constructHouse() {
      houseBuilder.buildBasic();
      houseBuilder.buildWalls();
      houseBuilder.roofed();
      return houseBuilder.buildHouse();
   }
```

###### 2.4.1 StringBuilder 中的建造者模式

- Appendable 接口定义了多个 append 方法(抽象方法), 即 Appendable 为抽象建造者, 定义了抽象方法
- AbstractStringBuilder 实现了 Appendable 接口方法，这里的 AbstractStringBuilder 已经是建造者，只是不能
  实例化
- StringBuilder 即充当了指挥者角色，同时充当了具体的建造者，建造方法的实现是由 AbstractStringBuilder 完
  成, 而 StringBuilder 继承了 AbstractStringBuilder

###### 2.4.2 建造者模式的注意事项和细节

- 客户端(使用程序)不必知道产品内部组成的细节，将产品本身与产品的创建过程解耦，使得相同的创建过程可
  以创建不同的产品对象
- 每一个具体建造者都相对独立，而与其他的具体建造者无关，因此可以很方便地替换具体建造者或增加新的具
  体建造者， 用户使用不同的具体建造者即可得到不同的产品对象

- 可以更加精细地控制产品的创建过程 。将复杂产品的创建步骤分解在不同的方法中，使得创建过程更加清晰，
  也更方便使用程序来控制创建过程
- 增加新的具体建造者无须修改原有类库的代码，指挥者类针对抽象建造者类编程，系统扩展方便，符合“开闭
  原则
- 建造者模式所创建的产品一般具有较多的共同点，其组成部分相似，如果产品之间的差异性很大，则不适合使
  用建造者模式，因此其使用范围受到一定的限制
- 如果产品的内部变化复杂，可能会导致需要定义很多具体建造者类来实现这种变化，导致系统变得很庞大，因
  此在这种情况下，要考虑是否选择建造者模式
- 抽象工厂模式 VS 建造者模式
  - 抽象工厂模式实现对产品家族的创建，一个产品家族是这样的一系列产品：具有不同分类维度的产品组合，采
    用抽象工厂模式不需要关心构建过程，只关心什么产品由什么工厂生产即可。而建造者模式则是要求按照指定
    的蓝图建造产品，它的主要目的是通过组装零配件而产生一个新产品

##### 2.5 适配器模式

- 适配器模式(Adapter Pattern)将某个类的接口转换成客户端期望的另一个接口表示， 主的目的是兼容性，让原本
  因接口不匹配不能一起工作的两个类可以协同工作。其别名为包装器(Wrapper)

- 适配器模式属于结构型模式
- 主要分为三类：类适配器，对象适配器模式，接口适配器模式

工作原理

- 适配器模式：将一个类的接口转换成另一种接口.让原本接口不兼容的类可以兼容

- 从用户的角度看不到被适配者，是解耦的

- 用户调用适配器转化出来的目标接口方法，适配器再调用被适配者的相关接口方法

- 用户收到反馈结果，感觉只是和目标接口交互

##### 2.6 桥接模式

桥接模式基本介绍：将实现与抽象放在两个不同的类层次中，使两个层次可以独立改变。

Bridge 模式基于类的最小设计原则，通过使用封装、聚合及继承等行为让不同的类承担不同的职责。它的主要
特点是把抽象(Abstraction)与行为实现(Implementation)分离开来，从而可以保持各部分的独立性以及应对他们的
功能扩展

桥接模式的注意事项：

实现了抽象和实现部分的分离，从而极大的提供了系统的灵活性，让抽象部分和实现部分独立开来，这有助于
系统进行分层设计，从而产生更好的结构化系统。

对于系统的高层部分，只需要知道抽象部分和实现部分的接口就可以了，其它的部分由具体业务来完成。

桥接模式替代多层继承方案，可以减少 子类的个数，降低系统的管理和维护成本