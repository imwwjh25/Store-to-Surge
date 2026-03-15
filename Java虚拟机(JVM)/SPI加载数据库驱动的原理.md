# 先一句话结论（背下来）

1. **`java.sql.Driver` 接口** → 由 **Bootstrap 类加载器** 加载（rt.jar 里）
2. **MySQL 的 Driver 实现类** → 放在项目里，由 **AppClassLoader** 加载
3. **Bootstrap 根本找不到项目里的类**，所以必须用 **线程上下文类加载器 (Thread.getContextClassLoader ())** 去加载实现类
4. **这就打破了双亲委派** —— 让父类加载器能使用子类加载器，违背了 “向上委托”

------

# 一、为什么 SPI 必须打破双亲委派？（核心痛点）

### 双亲委派规则：

**子类加载器才能看见父类加载的类，父类加载器看不见子类加载的类。**

### 真实场景：

- `java.sql.Driver`（接口）→ Bootstrap 加载
- `com.mysql.cj.jdbc.Driver`（实现）→ App 加载

### 问题来了：

**Bootstrap 加载的 DriverManager 要找 MySQL 的实现类，但它根本找不到！**

因为 Bootstrap 是父加载器，**无法访问子类加载器（App）加载的类**。

### 解决方案：

Java 设计了 **线程上下文类加载器（TCCL）**

让 **父类加载器可以反向使用子类加载器**，这就是**打破双亲委派**。

------

# 二、完整流程（面试口述版）

1. 项目引入 MySQL 驱动
2. `DriverManager`（Bootstrap 加载）初始化
3. 它调用 `ServiceLoader.load(Driver.class)`
4. **ServiceLoader 使用线程上下文类加载器（AppClassLoader）**
5. 去 classpath 下找到 `com.mysql.cj.jdbc.Driver`
6. 加载、注册驱动
7. **父加载器使用子加载器加载类 → 打破双亲委派**

------

# 三、底层代码拆解（JDK 源码 + 关键行）

## 1. 我们写的代码

java



运行









```
Connection conn = DriverManager.getConnection(url, user, pwd);
```

## 2. DriverManager 静态代码块（JDK 源码）

java



运行









```
// java.sql.DriverManager
static {
    loadInitialDrivers();
}

private static void loadInitialDrivers() {
    // 关键点！！！
    ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
    Iterator<Driver> driversIterator = loadedDrivers.iterator();

    while (driversIterator.hasNext()) {
        driversIterator.next(); // 这里会找到 MySQL Driver
    }
}
```

## 3. 最关键一句：ServiceLoader.load

java



运行









```
public static <S> ServiceLoader<S> load(Class<S> service) {
    // 获取 线程上下文类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();

    return ServiceLoader.load(service, cl);
}
```

### ✅ 这里就是**打破双亲委派的核心**

```
Thread.currentThread().getContextClassLoader()
```

**默认是 AppClassLoader！**

所以：

- **Bootstrap 加载的 DriverManager**
- 使用 **App 类加载器**
- 去加载 MySQL 驱动

**父加载器用子加载器 → 打破双亲委派！**

------

# 四、用一句话讲清楚 “打破”

双亲委派本来是：

**子类加载器 → 交给父类加载器**

现在变成：

**父类加载器 → 调用子类加载器**

**方向反了 → 打破双亲委派！**

------

# 五、面试官如果继续问：

### 问：为什么不能直接用 Bootstrap 加载 MySQL 驱动？

答：MySQL 驱动不在 `rt.jar` 里，Bootstrap 根本找不到。

### 问：那为什么不直接让 DriverManager 用 AppClassLoader？

答：`DriverManager` 是核心类，必须放 `rt.jar`，必须由 Bootstrap 加载。

### 问：那怎么解决？

答：**线程上下文类加载器**，让父加载器可以拿到子加载器，反向加载。

### 问：这算不算破坏双亲委派？

答：**算！**

双亲委派是向上委托，这里是**向下使用子类加载器**，所以是**标准打破双亲委派**。

------

# 六、终极面试满分回答（直接背）

JDBC 的 SPI 机制之所以打破双亲委派，原因如下：

1. `java.sql.Driver` 接口和 `DriverManager` 都在 **rt.jar** 中，由 **Bootstrap 类加载器** 加载。
2. 而 MySQL、Oracle 这些驱动实现类，在项目的 **classpath** 下，由 **AppClassLoader** 加载。
3. 根据双亲委派模型，**父类加载器无法访问子类加载器的类**，所以 `DriverManager` 找不到驱动。
4. 所以 JDK 使用 **线程上下文类加载器（Thread.getContextClassLoader ()）**，它默认是 **AppClassLoader**。
5. 让 Bootstrap 加载的 `DriverManager`，借助 **AppClassLoader** 去加载驱动实现。
6. **父类加载器使用子类加载器，方向相反 → 打破双亲委派模型。**

------

### 总结（最核心 3 句）

1. **接口：Bootstrap 加载**
2. **实现：App 加载**
3. **用线程上下文类加载器让父类加载器调用子类加载器 → 打破双亲委派**
