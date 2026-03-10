### 一、Java 8（2014 年，里程碑版本）

Java 8 是划时代的版本，引入的核心特性至今仍是 Java 开发的基础。

#### 核心特性

1. **Lambda 表达式**：简化匿名内部类的编写，让代码更简洁，支持函数式编程。

   

   java

   

   运行

   

   

   

   

   ```
   // 传统匿名内部类
   Runnable r1 = new Runnable() {
       @Override
       public void run() {
           System.out.println("传统方式");
       }
   };
   
   // Lambda 表达式
   Runnable r2 = () -> System.out.println("Lambda 方式");
   ```

   

2. **Stream API**：对集合进行高效的流式操作（过滤、映射、聚合等），支持并行处理。

   

   java

   

   运行

   

   

   

   

   ```
   List<String> list = Arrays.asList("Java8", "Java11", "Java17", "Java21");
   // 过滤长度为5的元素并输出
   list.stream()
       .filter(s -> s.length() == 5)
       .forEach(System.out::println); // 输出：Java11、Java17、Java21
   ```

   

3. **Optional 类**：解决空指针异常（NPE），优雅处理 null 值。

   

   java

   

   运行

   

   

   

   

   ```
   String str = null;
   // 安全获取值，避免 NPE，无值时返回默认值
   String result = Optional.ofNullable(str).orElse("默认值");
   System.out.println(result); // 输出：默认值
   ```

   

4. **接口默认方法 / 静态方法**：接口可以包含有实现的方法，避免接口升级导致所有实现类修改。

   

   java

   

   运行

   

   

   

   

   ```
   interface MyInterface {
       // 默认方法
       default void sayHello() {
           System.out.println("Hello Java8");
       }
       
       // 静态方法
       static void staticMethod() {
           System.out.println("接口静态方法");
       }
   }
   ```

   

5. **Date/Time API（JSR 310）**：替代老旧的 `Date`、`Calendar`，解决线程安全问题，API 更清晰。

   

   java

   

   运行

   

   

   

   

   ```
   // 获取当前日期
   LocalDate now = LocalDate.now();
   // 计算10天后的日期
   LocalDate after10Days = now.plusDays(10);
   System.out.println(after10Days);
   ```

   

### 二、Java 11（2018 年，LTS）

Java 11 是继 8 后的第一个 LTS 版本，聚焦**简化开发、性能提升、移除冗余特性**。

#### 核心特性

1. **局部变量类型推断（var 关键字）**：无需显式声明局部变量类型，编译器自动推断，简化代码。

   

   java

   

   运行

   

   

   

   

   ```
   // 无需写 String，编译器自动推断
   var str = "Java11";
   // 无需写 List<String>
   var list = Arrays.asList("a", "b", "c");
   ```

   

   注意：`var` 仅适用于**局部变量**，不能用于类成员、方法参数 / 返回值。

   

2. **简化的 HTTP Client API**：内置支持 HTTP/2 和 WebSocket，替代老旧的 `HttpURLConnection`，使用更便捷。

   

   java

   

   运行

   

   

   

   

   ```
   // 同步请求示例
   HttpClient client = HttpClient.newHttpClient();
   HttpRequest request = HttpRequest.newBuilder()
           .uri(URI.create("https://www.baidu.com"))
           .GET()
           .build();
   // 发送请求并获取响应
   HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
   System.out.println(response.statusCode()); // 输出 200
   ```

   

3. **ZGC 垃圾收集器（实验性）**：低延迟垃圾收集器，暂停时间不超过 10ms，适合大内存、低延迟场景。

   

4. **移除冗余特性**：如移除 `Java EE`、`CORBA` 模块，简化 JDK 体积；同时永久移除 `Nashorn` 脚本引擎（Java 15 彻底移除）。

   

5. **单行运行 Java 代码**：无需编译，直接运行单个 Java 文件（适合小脚本）。

   

   bash

   

   运行

   

   

   

   

   ```
   # 新建 Test.java，内容：public class Test { public static void main(String[] args) { System.out.println("Java11"); } }
   java Test.java # 直接运行，无需 javac 编译
   ```

   

### 三、Java 17（2021 年，LTS）

Java 17 是目前使用最广泛的 LTS 版本，聚焦**稳定性、安全性、简化开发**，同时引入了多个预览特性转正。

#### 核心特性

1. **密封类（Sealed Classes）**：限制类的继承 / 接口的实现，增强代码的可维护性。

   

   java

   

   运行

   

   

   

   

   ```
   // 密封类：仅允许 Cat、Dog 继承
   sealed class Animal permits Cat, Dog {}
   final class Cat extends Animal {} // 最终类，不能再被继承
   final class Dog extends Animal {}
   // 错误：Pig 未在 permits 中声明，无法继承 Animal
   // class Pig extends Animal {}
   ```

   

2. **Switch 表达式（正式版）**：支持返回值，简化 switch 语法，替代传统的 break + 变量赋值。

   

   java

   

   运行

   

   

   

   

   ```
   String day = "MON";
   // 传统 switch
   int num1 = 0;
   switch (day) {
       case "MON": num1 = 1; break;
       case "TUE": num1 = 2; break;
       default: num1 = 0;
   }
   
   // Java 17 Switch 表达式（带返回值）
   int num2 = switch (day) {
       case "MON" -> 1;
       case "TUE" -> 2;
       default -> 0;
   };
   System.out.println(num2); // 输出 1
   ```

   

3. **增强的空指针异常提示（NPE）**：异常信息会明确指出哪个变量为 null，快速定位问题。

   

   java

   

   运行

   

   

   

   

   ```
   String str = null;
   // 异常信息会显示：Cannot invoke "String.length()" because "str" is null
   System.out.println(str.length());
   ```

   

4. **Pattern Matching for instanceof（正式版）**：简化 instanceof 后的类型转换，减少冗余代码。

   

   java

   

   运行

   

   

   

   

   ```
   Object obj = "Java17";
   // 传统方式：先判断，再强制转换
   if (obj instanceof String) {
       String s = (String) obj;
       System.out.println(s.length());
   }
   
   // Java 17 简化方式：判断+转换一步完成
   if (obj instanceof String s) {
       System.out.println(s.length()); // 直接使用 s
   }
   ```

   

5. **ZGC 转正（正式版）**：Java 11 中实验性的 ZGC 变为正式特性，低延迟、高吞吐量，适合生产环境。

   

6. **移除废弃的 Applet API**：彻底移除过时的 Applet 相关类，简化 JDK。

   

### 四、Java 21（2023 年，最新 LTS）

Java 21 是目前最新的 LTS 版本，聚焦**性能提升、语法简化、新垃圾收集器**。

#### 核心特性

1. **虚拟线程（Virtual Threads，正式版）**：轻量级线程，替代传统的 OS 线程，极大降低高并发场景的资源消耗。

   

   java

   

   运行

   

   

   

   

   ```
   // 创建并启动虚拟线程（方式1）
   Thread.startVirtualThread(() -> {
       System.out.println("虚拟线程执行");
       try {
           Thread.sleep(1000); // 模拟耗时操作
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
   });
   
   // 方式2：通过 ExecutorService 创建
   try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
       // 提交10000个虚拟线程，资源消耗远低于OS线程
       for (int i = 0; i < 10000; i++) {
           executor.submit(() -> {
               Thread.sleep(1000);
               return i;
           });
       }
   }
   ```

   

   核心优势：虚拟线程由 JVM 管理，而非操作系统，创建百万级虚拟线程也不会耗尽系统资源，适合 IO 密集型场景（如微服务、接口调用）。

   

2. **Record 模式匹配（预览转正）**：简化 Record 类的字段提取，结合 instanceof 更高效。

   

   java

   

   运行

   

   

   

   

   ```
   // 定义 Record 类
   record Point(int x, int y) {}
   
   Object obj = new Point(10, 20);
   // 模式匹配：直接提取 x、y
   if (obj instanceof Point(int x, int y)) {
       System.out.println("x=" + x + ", y=" + y); // 输出 x=10, y=20
   }
   ```

   

3. **Sequenced Collections（有序集合）**：新增 `SequencedCollection`、`SequencedSet`、`SequencedMap` 接口，统一处理有序集合的首尾操作。

   

   java

   

   运行

   

   

   

   

   ```
   // List 实现 SequencedCollection，新增 reversed() 反转
   List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
   List<String> reversed = list.reversed();
   System.out.println(reversed); // 输出 [c, b, a]
   
   // Map 新增 reversed() 反转键值对顺序
   Map<String, Integer> map = new LinkedHashMap<>();
   map.put("Java8", 8);
   map.put("Java11", 11);
   Map<String, Integer> reversedMap = map.reversed();
   System.out.println(reversedMap); // 输出 {Java11=11, Java8=8}
   ```

   

4. **Shenandoah GC 转正（正式版）**：低暂停时间垃圾收集器，适合大内存、高吞吐量场景，与 ZGC 形成互补。

   

5. **String 模板（预览版）**：简化字符串拼接，替代 `String.format` 和拼接符 `+`，支持动态插入变量。

   

   java

   

   运行

   

   

   

   

   ```
   // 预览特性：需启用 --enable-preview
   String name = "Java21";
   int version = 21;
   // 字符串模板，直接嵌入变量
   String str = STR."Hello \{name}, version is \{version}";
   System.out.println(str); // 输出 Hello Java21, version is 21
   ```

   

### 总结

表格







|  版本   |        核心定位        |                最关键特性                 |
| :-----: | :--------------------: | :---------------------------------------: |
| Java 8  |     函数式编程基础     | Lambda、Stream API、Optional、新日期 API  |
| Java 11 | 首个现代 LTS、简化开发 |   var 关键字、HTTP Client、ZGC（实验）    |
| Java 17 |   稳定 LTS、语法简化   | 密封类、Switch 表达式、ZGC 转正、增强 NPE |
| Java 21 |  最新 LTS、高性能并发  |    虚拟线程、Record 模式匹配、有序集合    |

### 关键点回顾

1. **LTS 版本优先级**：生产环境优先选择 11、17、21（长期支持），Java 8 虽经典但已逐步淘汰。
2. **核心效率特性**：Java 8 的 Stream/Lambda 简化集合操作，Java 21 的虚拟线程大幅提升并发性能。
3. **语法简化趋势**：从 Java 11 的 var，到 17 的 switch 表达式，再到 21 的 Record 模式匹配，Java 持续降低编码冗余。
