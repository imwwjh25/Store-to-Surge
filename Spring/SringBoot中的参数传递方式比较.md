**SpringBoot 接口传参 = URL 路径上 + 请求头里 + 请求体里，三种位置，五种写法。**

------

## 一、先分清 3 个核心位置（关键！）

1. **路径变量**：写在 URL 路径里 `/user/1001`
2. **查询参数**：写在 URL ? 后面 `/user?id=1001`
3. **请求体参数**：放在 POST 请求体里（JSON / 表单）

------

## 二、5 种传参方式 + 注解 彻底对比

### 1. **@PathVariable（路径变量 / Path 参数）**

**放在 URL 路径里**

plaintext











```
GET /user/1001
```

**注解**：`@PathVariable`

**示例**：

java



运行









```
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id)
```

**特点**：

- 参数是**URL 路径的一部分**
- 必须传，**不能为空**
- 用于定位资源（查某个用户、某个订单）
- 只支持 GET/DELETE/PUT

**适用场景**：查询、删除、修改某个明确资源

------

### 2. **@RequestParam（查询参数 / URL? 后参数）**

**放在 URL ? 后面**

plaintext











```
GET /user?id=1001&name=张三
```

**注解**：`@RequestParam`

**示例**：

java



运行









```
@GetMapping("/user")
public User getUser(
    @RequestParam Long id,
    @RequestParam(required = false) String name
)
```

**特点**：

- 拼接在 URL 问号后面
- 可以选传（`required=false`）
- 支持多参数
- GET、POST 都能用

**适用场景**：分页、筛选、查询条件

------

### 3. **@RequestBody（请求体参数）**

**放在 POST 请求体里（JSON 格式）**

**最常用！前后端分离必用**

json











```
POST /user
{
  "id":1001,
  "name":"张三"
}
```

**注解**：`@RequestBody`

**示例**：

java



运行









```
@PostMapping("/user")
public User addUser(@RequestBody User user)
```

**特点**：

- 传**复杂对象、数组、嵌套结构**
- 用 JSON 格式
- 只能用于 **POST / PUT**
- 一个接口只能有**一个** @RequestBody

**适用场景**：新增、修改、提交表单、复杂接口

------

### 4. **form 表单参数（不用写注解！）**

**表单提交，参数自动绑定**

plaintext











```
POST /login
username=admin&password=123
```

**代码**：

java



运行









```
@PostMapping("/login")
public String login(String username, String password)
```

**特点**：

- 不用写任何注解
- 简单键值对
- 不适合复杂对象

------

### 5. **@RequestHeader（请求头参数）**

**放在请求头里**

plaintext











```
Token: abc123
```

**代码**：

java



运行









```
@GetMapping("/user")
public User info(@RequestHeader("Token") String token)
```

------

## 三、最容易混淆的 3 个核心区别（面试必问）

### 1. **@RequestParam vs @PathVariable**

表格







|   特点   | @RequestParam | @PathVariable |
| :------: | :-----------: | :-----------: |
|   位置   |  URL ? 后面   |  URL 路径里   |
|   示例   |  /user?id=1   |    /user/1    |
| 是否必须 |  可设非必传   |  **必须传**   |
|   用途   |   查询条件    |   定位资源    |

### 2. **@RequestParam vs @RequestBody**

表格







|   特点   | @RequestParam |  @RequestBody   |
| :------: | :-----------: | :-------------: |
|   位置   |   URL 后面    | 请求体（Body）  |
|   格式   |    键值对     |    JSON 对象    |
|  复杂度  |   简单参数    | 复杂对象 / 数组 |
| 请求方式 |   GET/POST    |    POST/PUT     |

### 3. **什么时候用哪个？（最实用口诀）**

- **查单个资源** → **@PathVariable**  `/user/1001`
- **查列表 / 筛选** → **@RequestParam**  `/user?page=1&size=10`
- **新增 / 修改 / 提交** → **@RequestBody**（JSON）
- **表单登录** → 不用注解 或 @RequestParam
- **传文件** → form-data + @RequestParam

------

## 四、一个接口同时用三种（真实开发常用）

java



运行









```
@PostMapping("/user/{userId}/role")
public Result addRole(
    @PathVariable Long userId,         // 路径
    @RequestParam String role,         // URL参数
    @RequestBody RoleDto roleDto      // 请求体JSON
)
```

------

### 总结（最强记忆版）

1. **路径里** → `@PathVariable`（查单个）
2. **URL? 后面** → `@RequestParam`（筛选 / 分页）
3. **JSON 请求体** → `@RequestBody`（增删改）
4. **表单** → 不用注解
5. **请求头** → `@RequestHeader`

这就是 SpringBoot **全部传参方式**，100% 覆盖工作场景。