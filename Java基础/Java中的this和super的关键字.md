# Java 中 this 和 super 关键字 面试 / 学习完整解析

this 和 super 是 Java 中两个核心关键字，分别用于**当前对象**和**父类对象**的访问，我会从「核心定义→使用场景→语法规则→对比区别→实战示例」全维度讲解，帮你彻底理解并能灵活运用。

## 一、核心定义（先理清本质）




| 关键字 |                           核心含义                           |                           底层逻辑                           |
| :----: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  this  | 代表**当前对象的引用**（即当前正在执行方法 / 构造器的对象）  | 编译器会隐式给非静态方法传入 this 引用，指向调用该方法的对象实例 |
| super  | 代表**父类对象的引用**（用于访问父类的成员，突破子类对父类成员的隐藏） | 本质是访问子类对象中继承自父类的那部分成员，并非单独的父类对象实例 |

## 二、this 关键字的使用场景（4 类核心场景）

### 1. 区分成员变量和局部变量（最常用）

当方法 / 构造器的局部变量名与成员变量名相同时，用 `this.成员变量` 明确访问成员变量。






```
public class Person {
    private String name;
    private int age;

    // 构造器中区分局部变量和成员变量
    public Person(String name, int age) {
        this.name = name; // this.name 是成员变量，name 是局部变量
        this.age = age;
    }

    // 普通方法中区分
    public void setName(String name) {
        this.name = name;
    }
}
```

### 2. 调用当前类的构造器

用 `this(参数)` 调用当前类的其他构造器，**必须放在构造器的第一行**，且不能和 `super(参数)` 同时使用。




```
public class Person {
    private String name;
    private int age;

    // 无参构造器
    public Person() {
        this("默认名称", 18); // 调用有参构造器，必须在第一行
        System.out.println("无参构造器执行");
    }

    // 有参构造器
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        System.out.println("有参构造器执行");
    }
}

// 测试
public class Test {
    public static void main(String[] args) {
        Person p = new Person(); 
        // 输出：有参构造器执行 → 无参构造器执行
    }
}
```

### 3. 调用当前类的成员方法

用 `this.方法名()` 调用当前类的方法（可省略，但建议显式写，增强可读性）。








```
public class Person {
    public void eat() {
        System.out.println("吃饭");
    }

    public void live() {
        this.eat(); // 等价于 eat()，调用当前类的 eat 方法
        System.out.println("生活");
    }
}
```

### 4. 表示当前对象本身

可将 this 作为参数传递、作为返回值返回，明确指向当前对象。







```
public class Person {
    // 作为返回值，实现链式调用
    public Person getSelf() {
        return this; // 返回当前对象
    }

    // 作为参数传递
    public void printSelf(Person p) {
        System.out.println(p == this); // true，传入的是当前对象
    }

    public static void main(String[] args) {
        Person p = new Person();
        p.printSelf(p.getSelf()); // 输出 true
    }
}
```

## 三、super 关键字的使用场景（3 类核心场景）

### 1. 访问父类的成员变量

当子类定义了和父类同名的成员变量（隐藏），用 `super.父类变量名` 访问父类变量。



```
// 父类
class Parent {
    String name = "父类名称";
}

// 子类
class Child extends Parent {
    String name = "子类名称";

    public void showName() {
        System.out.println(name); // 子类名称（局部优先）
        System.out.println(super.name); // 父类名称（访问父类变量）
    }
}

// 测试
public class Test {
    public static void main(String[] args) {
        Child c = new Child();
        c.showName(); 
        // 输出：
        // 子类名称
        // 父类名称
    }
}
```

### 2. 调用父类的成员方法

当子类重写了父类的方法，用 `super.父类方法名()` 调用父类的原始方法。






```
// 父类
class Parent {
    public void say() {
        System.out.println("父类的say方法");
    }
}

// 子类
class Child extends Parent {
    @Override
    public void say() {
        super.say(); // 调用父类的say方法
        System.out.println("子类的say方法");
    }
}

// 测试
public class Test {
    public static void main(String[] args) {
        Child c = new Child();
        c.say();
        // 输出：
        // 父类的say方法
        // 子类的say方法
    }
}
```

### 3. 调用父类的构造器

用 `super(参数)` 调用父类的构造器，**必须放在子类构造器的第一行**，且不能和 `this(参数)` 同时使用。

> 注意：如果子类构造器中没有显式写 `super()`，编译器会自动在第一行添加 `super()`（调用父类无参构造器）。





```
// 父类
class Parent {
    // 无参构造器
    public Parent() {
        System.out.println("父类无参构造器");
    }

    // 有参构造器
    public Parent(String msg) {
        System.out.println("父类有参构造器：" + msg);
    }
}

// 子类
class Child extends Parent {
    // 子类无参构造器（编译器自动加 super()）
    public Child() {
        // super(); // 隐式存在
        System.out.println("子类无参构造器");
    }

    // 子类有参构造器（显式调用父类有参构造器）
    public Child(String msg) {
        super(msg); // 必须在第一行
        System.out.println("子类有参构造器：" + msg);
    }
}

// 测试
public class Test {
    public static void main(String[] args) {
        Child c1 = new Child();
        // 输出：
        // 父类无参构造器
        // 子类无参构造器

        Child c2 = new Child("测试");
        // 输出：
        // 父类有参构造器：测试
        // 子类有参构造器：测试
    }
}
```

## 四、this 和 super 的核心区别（必背）


|    维度    |                             this                             |             super             |
| :--------: | :----------------------------------------------------------: | :---------------------------: |
|  指向对象  |                        当前对象的引用                        |   子类对象中父类部分的引用    |
|  访问范围  |               访问当前类的成员（变量 / 方法）                | 访问父类的成员（变量 / 方法） |
| 构造器调用 |                    调用当前类的其他构造器                    |       调用父类的构造器        |
|  使用位置  |                可在非静态方法 / 构造器中使用                 | 可在非静态方法 / 构造器中使用 |
|  静态环境  |                不能在静态方法中使用（无对象）                |     不能在静态方法中使用      |
|  共存规则  | 构造器中 `this()` 不能和 `super()` 同时出现，且都必须在第一行 |                               |

## 五、常见易错点（避坑指南）

### 1. 静态方法中不能用 this/super

静态方法属于 “类”，而非 “对象”，执行时没有对象实例，因此 this/super 无意义，编译器直接报错。







```
public class Test {
    public static void staticMethod() {
        // this.print(); // 报错：Cannot use this in a static context
        // super.print(); // 报错：Cannot use super in a static context
    }

    public void print() {}
}
```

### 2. 父类无参构造器缺失导致编译失败

如果父类只定义了有参构造器，没有显式写无参构造器，子类构造器中默认的 `super()` 会找不到父类无参构造器，编译报错。









```
// 父类（只有有参构造器）
class Parent {
    public Parent(String msg) {}
}

// 子类（编译报错）
class Child extends Parent {
    public Child() {
        // 编译器自动加 super()，但父类无无参构造器 → 报错
    }
}
// 解决：子类构造器显式调用父类有参构造器
class ChildFix extends Parent {
    public ChildFix() {
        super("默认值"); // 显式调用父类有参构造器
    }
}
```

### 3. super 不能访问父类的 private 成员

private 成员仅在本类可见，子类无法通过 super 访问（需通过父类的 public/protected 方法间接访问）。








```
class Parent {
    private int age = 18;

    public int getAge() {
        return age; // 父类内部访问
    }
}

class Child extends Parent {
    public void showAge() {
        // System.out.println(super.age); // 报错：age has private access in Parent
        System.out.println(super.getAge()); // 正确：通过父类方法访问
    }
}
```

## 六、总结（核心要点回顾）

1. **this**：指向当前对象，核心用于「区分同名变量」「调用当前类构造器」「传递 / 返回当前对象」，仅能在非静态环境使用；
2. **super**：指向子类对象的父类部分，核心用于「访问父类被隐藏的成员」「调用父类构造器」，仅能在非静态环境使用；
3. **关键规则**：构造器中 `this()`/`super()` 必须在第一行，且不能共存；静态方法中禁用 this/super；
4. **易错点**：父类无参构造器缺失、访问父类 private 成员、静态环境使用 this/super。
