### 一、核心参数传递注解详解

#### 1. @PathVariable：路径变量（URL 路径中取值）

**作用**：从 URL 的**路径部分**提取参数（比如 `/user/123` 中的 `123`），适用于 RESTful 风格的接口。

**特点**：参数是 URL 路径的一部分，而非请求参数 / 请求体。

**代码示例**：







```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PathVariableController {
    // 接口地址：http://localhost:8080/user/123
    @GetMapping("/user/{userId}")
    public String getUserById(@PathVariable("userId") Long id) {
        // @PathVariable("userId") 绑定 URL 中 {userId} 对应的值到变量 id
        return "获取到的用户ID：" + id; // 输出：获取到的用户ID：123
    }

    // 可选参数（Spring Boot 2.0+）：添加 required = false，路径中可省略
    @GetMapping("/user/{userId}/role/{roleId}")
    public String getUserRole(
            @PathVariable Long userId, // 省略value时，变量名需和路径占位符一致
            @PathVariable(required = false) Long roleId) {
        return "用户ID：" + userId + "，角色ID：" + (roleId == null ? "无" : roleId);
    }
}
```

#### 2. @RequestParam：请求参数（URL 查询字符串 / 表单提交）

**作用**：从 URL 的**查询字符串**（`?key=value`）或 **表单提交**（`application/x-www-form-urlencoded`）中提取参数，适用于简单的键值对参数。

**特点**：参数是 URL 后缀的键值对，或表单字段，可指定是否必传、默认值。

**代码示例**：





```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RequestParamController {
    // 接口地址：http://localhost:8080/user?name=张三&age=20
    @GetMapping("/user")
    public String getUserInfo(
            @RequestParam("name") String userName, // 绑定name参数到userName
            @RequestParam(required = false, defaultValue = "18") Integer age) {
        // required = false：参数非必传；defaultValue：参数缺失时的默认值
        return "用户名：" + userName + "，年龄：" + age; // 输出：用户名：张三，年龄：20
    }
}
```

#### 3. @RequestBody：请求体（JSON/XML 等格式）

**作用**：从 HTTP 请求的**请求体**中读取数据，通常用于接收 JSON/XML 格式的复杂数据（如对象、数组），适用于 POST/PUT 等请求。

**特点**：只能绑定到一个参数（请求体只有一个），且请求头需指定 `Content-Type: application/json`（或对应格式）。

**代码示例**：






```
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

// 定义接收参数的实体类
class User {
    private String name;
    private Integer age;
    // 必须有无参构造器（Spring默认反射创建）、getter/setter
    public User() {}
    // getter/setter 省略，实际开发需补充
}

@RestController
public class RequestBodyController {
    // 接口地址：http://localhost:8080/user
    // 请求体：{"name":"张三","age":20}
    // 请求头：Content-Type: application/json
    @PostMapping("/user")
    public String addUser(@RequestBody User user) {
        // @RequestBody 将JSON请求体转换为User对象
        return "新增用户：" + user.getName() + "，年龄：" + user.getAge();
    }
}
```

#### 4. 补充注解（常用）

- `@RequestHeader`：从请求头中取值（比如 `@RequestHeader("token") String token`）；
- `@CookieValue`：从 Cookie 中取值（比如 `@CookieValue("JSESSIONID") String sessionId`）。

### 二、核心注解的区别对比




|     注解      |        参数位置        |         数据格式          |     请求方式      |            核心场景             |
| :-----------: | :--------------------: | :-----------------------: | :---------------: | :-----------------------------: |
| @PathVariable | URL 路径（/user/{id}） | 简单类型（字符串 / 数字） |  GET/POST/PUT 等  |   RESTful 接口（获取资源 ID）   |
| @RequestParam | URL 查询字符串 / 表单  |        简单键值对         | 主要 GET，也 POST |   简单参数筛选（分页、搜索）    |
| @RequestBody  |      HTTP 请求体       |   JSON/XML（复杂对象）    |   主要 POST/PUT   | 提交 / 修改复杂数据（创建对象） |

### 三、使用注意事项

1. `@RequestBody` 不能用于 GET 请求：GET 请求通常没有请求体，强行使用会导致参数绑定失败；
2. `@PathVariable` 注意 URL 编码：如果参数包含特殊字符（如空格、中文），需确保 URL 编码；
3. `@RequestParam` 可批量接收：用 `@RequestParam Map<String, String> params` 接收所有查询参数；
4. 混合使用：一个接口可同时使用多个注解（比如 `@PathVariable` + `@RequestParam` + `@RequestBody`）。

### 总结

1. **@PathVariable** 用于从 URL 路径中取参数，适配 RESTful 风格，核心是 “路径里的参数”；
2. **@RequestParam** 用于从 URL 查询字符串 / 表单中取简单键值对，核心是 “URL 后缀 / 表单的参数”；
3. **@RequestBody** 用于从请求体中取复杂 JSON/XML 数据，核心是 “请求体里的结构化数据”。

记住核心原则：简单 ID / 路径参数用 `@PathVariable`，简单筛选参数用 `@RequestParam`，复杂对象提交用 `@RequestBody`。