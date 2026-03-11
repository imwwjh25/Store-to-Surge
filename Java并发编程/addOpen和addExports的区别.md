在 Java 9 引入模块系统（Module System）后，`--add-opens` 和 `--add-exports` 都是用来**突破模块访问限制**的命令行参数，但二者的适用场景、访问权限、底层逻辑完全不同，也是你升级 Java 21 时配置 PowerMock/Surefire 必须理清的核心知识点。

先明确一个前提：Java 9+ 将所有核心类库（如 `java.base`）拆分为模块化，模块内的包默认遵循「对外只暴露 `export` 的包，且仅允许反射访问 `open` 的包」，未暴露的包无法被外部模块访问（包括反射）。

## 一、核心概念：模块的访问规则（铺垫）

要理解两个参数的区别，先记住模块的 2 个核心访问控制规则：

1. **编译 / 运行时直接访问**：模块只能访问其他模块显式 `exports` 的包；
2. **反射访问**：即使包被 `exports`，反射访问该包内的私有成员（如私有方法、字段）仍会被禁止，需模块显式 `opens` 该包。

简单类比：

- 模块是一个「房子」，包是「房间」；
- `exports` 是「把房间的门打开，允许外人直接走进来（直接访问）」；
- `opens` 是「把房间的墙拆了，允许外人从任何角度窥探（反射访问）」。

## 二、`--add-exports`：开放「编译 / 运行时直接访问权限」

### 1. 定义

```
--add-exports <模块>/<包>=<目标模块列表>
```

强制让指定模块的指定包，向目标模块**开放「编译 / 运行时的直接访问权限」**（等价于模块内用 `exports <包>` 声明）。

### 2. 核心特点

- **权限范围**：仅允许「直接访问」（如调用公共方法、访问公共字段），不允许反射访问私有 / 受保护成员；
- **生效阶段**：编译期 + 运行期；
- **适用场景**：外部模块需要直接调用某个模块中未导出（`exports`）的包内的**公共成员**。

### 3. 示例

bash



运行









```
# 让 java.base 模块的 java.lang.reflect 包，向所有未命名模块开放直接访问权限
java --add-exports java.base/java.lang.reflect=ALL-UNNAMED -jar your-app.jar
```

场景：你的代码（未模块化，属于 `ALL-UNNAMED`）需要直接调用 `java.lang.reflect` 包中的公共类（如 `Method`），但该包未被 `java.base` 显式导出时，用此参数。

## 三、`--add-opens`：开放「反射访问权限」

### 1. 定义

```
--add-opens <模块>/<包>=<目标模块列表>
```

强制让指定模块的指定包，向目标模块**开放「反射访问权限」**（等价于模块内用 `opens <包>` 声明）。

### 2. 核心特点

- **权限范围**：不仅允许直接访问公共成员，还允许通过反射访问包内的**所有成员**（包括私有方法、私有字段、受保护成员）；
- **生效阶段**：仅运行期（编译期不生效）；
- **适用场景**：外部模块需要通过反射访问某个模块中未开放（`opens`）的包内的非公共成员（如 PowerMock 反射 Mock 私有方法、Lombok 生成私有字段的 getter/setter）。

### 3. 示例

bash



运行









```
# 让 java.base 模块的 java.lang 包，向所有未命名模块开放反射访问权限
java --add-opens java.base/java.lang=ALL-UNNAMED -jar your-app.jar
```

场景：PowerMock 需要反射修改 `java.lang.String` 的私有字段，或 Mock `java.lang` 包内的静态方法时，必须用此参数（这也是你配置 PowerMock 时核心用到的参数）。

## 四、核心区别对比表

表格







|       维度       |        `--add-exports`         |               `--add-opens`                |
| :--------------: | :----------------------------: | :----------------------------------------: |
|   **核心权限**   | 开放「直接访问权」（公共成员） |   开放「反射访问权」（所有成员，含私有）   |
|   **生效阶段**   |        编译期 + 运行期         |                  仅运行期                  |
| **底层等价声明** |     模块内 `exports <包>`      |            模块内 `opens <包>`             |
|   **访问深度**   |  仅能访问 `public` 修饰的成员  | 可访问 `public`/`private`/`protected` 成员 |
| **典型使用场景** |  调用未导出包的公共类 / 方法   |  反射访问私有成员（PowerMock/Lombok/ASM）  |
|   **权限强度**   |    弱（仅开放最小必要权限）    |           强（突破所有访问限制）           |

## 五、实战场景：什么时候用哪个？

### 1. 用 `--add-exports` 的场景

- 代码编译时报 `package xxx is not visible`（包不可见）；

- 运行时直接调用某个模块的包内公共方法，报 `NoClassDefFoundError`/`IllegalAccessError`；

- 示例：

  bash

  

  运行

  

  

  

  

  ```
  # 解决编译/运行时访问 java.sql 未导出包的问题
  javac --add-exports java.sql/com.mysql.cj=my.module Main.java
  ```

  

### 2. 用 `--add-opens` 的场景

- 反射调用时报 `IllegalAccessException`（非法访问）；

- PowerMock/Mockito 反射 Mock 私有方法 / 静态方法失败；

- Lombok 生成的 getter/setter 无法访问私有字段；

- 示例（你配置 PowerMock 时的核心参数）：

  bash

  

  运行

  

  

  

  

  ```
  java --add-opens java.base/java.lang=ALL-UNNAMED \
       --add-opens java.base/java.util=ALL-UNNAMED \
       -jar your-test.jar
  ```

  

### 3. 特殊值 `ALL-UNNAMED`

两个参数中都用到的 `ALL-UNNAMED` 表示「所有未模块化的代码」（即传统的 jar 包、未定义 `module-info.java` 的项目代码），这是升级时最常用的目标模块（因为大部分应用还未适配模块化）。

## 六、关键注意事项

1. **最小权限原则**：尽量避免用 `ALL-UNNAMED`，而是指定具体的目标模块（如 `--add-opens java.base/java.lang=com.your.project`），减少权限泄露；
2. **Java 版本差异**：Java 17+/21 对访问限制更严格，`--add-opens` 是解决 PowerMock 反射问题的**唯一方案**，`--add-exports` 无法替代；
3. **不要混用**：仅当同时需要「直接访问」+「反射访问」时，才需要同时配置两个参数，否则按需选择即可（比如 PowerMock 场景只需 `--add-opens`）。

## 总结

1. `--add-exports` 解决「模块间直接访问未导出包的公共成员」问题，生效于编译 + 运行期，权限弱；
2. `--add-opens` 解决「反射访问模块内私有成员」问题，仅生效于运行期，权限强，是 Java 21 配置 PowerMock/Surefire 的核心参数；
3. 升级时的核心场景：PowerMock 反射访问 `java.base` 核心包 → 用 `--add-opens`；普通代码访问未导出包的公共类 → 用 `--add-exports`。
