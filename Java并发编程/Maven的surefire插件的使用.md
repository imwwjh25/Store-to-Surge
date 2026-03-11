### Maven Surefire 插件全解析（核心功能 + 使用场景 + 避坑指南）

Maven Surefire 插件是 Maven 生态中**专门用于执行单元测试的核心插件**，也是 Java 项目构建中不可或缺的组件。无论是普通单元测试（JUnit/Jupiter）、集成测试，还是从 Java 8 升级到 Java 21 的适配，都需要掌握它的配置和调优方式。

下面从「核心功能」「基础配置」「Java 版本适配（重点）」「常见问题 & 避坑」四个维度详细讲解。

## 一、核心功能

Surefire 插件的核心目标是：**在 Maven 构建的 `test` 阶段自动执行项目中的单元测试，并生成测试报告**，具体能力包括：

1. **自动识别测试类**：默认扫描 `src/test/java` 下符合命名规则的类（如 `*Test.java`、`Test*.java`、`*TestCase.java`）；
2. **适配主流测试框架**：支持 JUnit 3/4/5（Jupiter）、TestNG 等，无需额外配置即可兼容；
3. **测试隔离与并发**：支持多线程执行测试（提升效率），且每个测试用例运行在独立的 JVM 中（避免用例间污染）；
4. **测试结果管控**：可配置「测试失败是否终止构建」「忽略特定测试用例」等；
5. **报告生成**：默认生成 XML/HTML 格式的测试报告（路径：`target/surefire-reports`）。

## 二、基础配置（POM.xml）

### 1. 最简配置（Java 8 场景）

xml











```
<build>
    <plugins>
        <!-- Surefire 插件基础配置 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>2.22.2</version> <!-- Java 8 常用稳定版本 -->
            <configuration>
                <!-- 核心参数：指定测试框架（JUnit 4 示例） -->
                <providerRef>junit4</providerRef>
                <!-- 可选：跳过测试（打包时常用） -->
                <!-- <skipTests>true</skipTests> -->
                <!-- 可选：多线程执行测试（线程数=CPU核心数） -->
                <threadCount>4</threadCount>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 2. Java 21 适配配置（重点）

从 Java 8 升级到 Java 21 时，Surefire 插件的版本和配置必须同步升级，否则会出现「测试无法运行」「类版本不兼容」等问题：

xml











```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version> <!-- 必须 ≥3.0.0-M5 才支持 Java 21 -->
    <configuration>
        <!-- 适配 Java 21 模块化/字节码版本 -->
        <jdkToolchain>
            <version>21</version>
        </jdkToolchain>
        <!-- JUnit 5（Jupiter）配置（Java 21 推荐） -->
        <providerRef>junit-platform</providerRef>
        <!-- 解决 Java 21 反射访问限制问题 -->
        <argLine>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.util=ALL-UNNAMED
        </argLine>
        <!-- 可选：指定仅执行特定测试用例 -->
        <includes>
            <include>**/*Test.java</include>
        </includes>
        <!-- 可选：排除不需要执行的测试用例 -->
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
    </configuration>
    <!-- 依赖：JUnit 5 引擎（必须） -->
    <dependencies>
        <dependency>
            <groupId>org.junit.platform</groupId>
            <artifactId>junit-platform-surefire-provider</artifactId>
            <version>1.10.0</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.10.0</version>
        </dependency>
    </dependencies>
</plugin>
```

### 关键版本对应关系（避免兼容性问题）

表格







| Java 版本 | Surefire 最低版本 | 适配的 JUnit 版本 |
| :-------: | :---------------: | :---------------: |
|  Java 8   |      2.22.2       |   JUnit 4/5.5.x   |
|  Java 17  |     3.0.0-M7      |   JUnit 5.8.x+    |
|  Java 21  |      3.2.0+       |   JUnit 5.9.x+    |

## 三、常用命令

bash



运行









```
# 执行所有测试（Maven test 阶段）
mvn test

# 跳过测试打包
mvn clean package -DskipTests

# 强制跳过测试（包括集成测试）
mvn clean install -Dmaven.test.skip=true

# 仅执行指定测试类
mvn test -Dtest=UserServiceTest

# 仅执行指定测试方法
mvn test -Dtest=UserServiceTest#testCreateUser
```

## 四、常见问题 & 避坑点

### 1. 问题 1：Java 21 运行测试报 `Unsupported class file major version 65`

- **原因**：Surefire 插件版本过低，无法识别 Java 21 的字节码版本（65）；
- **解决**：升级 Surefire 到 3.2.0+ 版本（推荐 3.2.5）。

### 2. 问题 2：测试中反射访问 Java 核心类报 `IllegalAccessException`

- **原因**：Java 9+ 加强了模块访问限制，测试代码反射访问 `java.lang`/`java.util` 等模块被拒绝；
- **解决**：在 `argLine` 中添加 `--add-opens` 参数（参考上面的 Java 21 配置）。

### 3. 问题 3：JUnit 5 测试用例未执行（无报错，测试数为 0）

- 原因

  ：

  1. 未引入 `junit-jupiter-engine` 依赖；
  2. 测试类未加 `@Test` 注解（JUnit 5 需用 `org.junit.jupiter.api.Test`，而非 JUnit 4 的 `org.junit.Test`）；

  

- 解决

  ：

  - 补充 JUnit 5 引擎依赖；
  - 替换注解为 JUnit 5 版本。

  

### 4. 问题 4：多线程执行测试时出现随机失败

- **原因**：测试用例之间共享了静态资源（如数据库连接、全局变量），多线程下出现竞态条件；

- 解决

  ：

  1. 为每个测试用例创建独立的资源实例（如 `@BeforeEach` 初始化、`@AfterEach` 销毁）；

  2. 降低线程数，或添加 

     ```
     parallel=none
     ```

      禁用并发：

     xml

     

     

     

     

     

     ```
     <configuration>
         <parallel>none</parallel>
     </configuration>
     ```

     

  

### 5. 问题 5：测试报告乱码（中文显示异常）

- **原因**：Surefire 默认编码不是 UTF-8；

- 解决

  ：添加编码配置：

  xml

  

  

  

  

  

  ```
  <configuration>
      <encoding>UTF-8</encoding>
      <inputEncoding>UTF-8</inputEncoding>
      <outputEncoding>UTF-8</outputEncoding>
  </configuration>
  ```

  

## 五、进阶用法：集成测试分离

Surefire 用于单元测试，而 `maven-failsafe-plugin`（Surefire 兄弟插件）专门用于集成测试，两者配合可实现「单元测试 + 集成测试分离执行」：

xml











```
<!-- 集成测试配置（Failsafe 插件） -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

- 执行单元测试：`mvn test`；
- 执行集成测试：`mvn verify`。

### 总结

1. Maven Surefire 是执行单元测试的核心插件，Java 21 升级时必须将其版本升级到 3.2.0+，并适配 JDK 工具链和模块访问参数；
2. 核心避坑点：版本匹配（Java 版本 ↔ Surefire 版本 ↔ JUnit 版本）、反射访问限制、多线程测试资源隔离；
3. 进阶场景可使用 Surefire + Failsafe 分离单元测试和集成测试，提升构建效率。
