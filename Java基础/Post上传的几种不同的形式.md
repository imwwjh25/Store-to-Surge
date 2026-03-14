# POST 请求参数传递的 4 种核心方式（最全 + 实用）

POST 请求的参数**不是拼在 URL 里**，而是放在**请求体（Request Body）\**中传递，后端通过不同的\**`Content-Type`** 解析参数，这是最核心的区别。

日常开发中最常用的有 **4 种标准方式**，我给你整理成最易懂的版本：

------

## 1. form-urlencoded（最常用、默认格式）

**全称**：`application/x-www-form-urlencoded`

**特点**：

- 表单默认提交方式，**键值对**格式
- 参数会被**URL 编码**（空格变 +、特殊字符转码）
- 不适合传**大文件 / 二进制数据**
- 所有后端框架天然支持

**请求示例**：









```
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin&password=123456
```

**适用场景**：登录、查询、普通表单提交。

------

## 2. form-data（传文件必用）

**全称**：`multipart/form-data`

**特点**：

- **唯一支持上传文件**的格式
- 可以同时传**文本参数 + 文件**
- 不会对参数编码，效率高
- 前端表单加 `enctype="multipart/form-data"`

**请求示例**：











```
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="username"

admin
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="test.jpg"
Content-Type: image/jpeg

(文件二进制数据)
```

**适用场景**：上传图片、视频、Excel、任何文件。

------

## 3. raw（纯文本 / JSON/XML）

**最主流的接口格式**，前端、后端接口联调首选

**常见子类型**：

- `application/json`（**90% 后端接口用这个**）
- `text/plain`（纯文本）
- `application/xml`（XML 格式）

**JSON 格式请求示例**：







```
POST /api/user HTTP/1.1
Content-Type: application/json

{
  "username": "admin",
  "age": 20,
  "hobby": ["coding", "reading"]
}
```

**特点**：

- 支持**复杂嵌套对象、数组**
- 结构清晰，前后端最友好
- 无法直接传文件

**适用场景**：前后端分离接口、RESTful API、微服务通信。

------

## 4. binary（二进制流）

**全称**：`application/octet-stream`

**特点**：

- 直接把**二进制数据**放在请求体
- 没有键值对，整个 body 就是一个文件
- 适合传**单个文件 / 字节流**

**适用场景**：大文件流式上传、图片 / 音频裸数据传输。

------

# 快速对比（一眼选对）







|      方式       |            Content-Type             |   核心特点   |     适用场景     |
| :-------------: | :---------------------------------: | :----------: | :--------------: |
| form-urlencoded | `application/x-www-form-urlencoded` | 键值对、编码 | 普通表单、无文件 |
|    form-data    |        `multipart/form-data`        |   支持文件   |   **上传文件**   |
|    raw(json)    |         `application/json`          |   复杂结构   | **接口联调首选** |
|     binary      |     `application/octet-stream`      |   纯二进制   |   单文件流上传   |

------

# 补充：容易混淆的知识点

1. **GET 和 POST 的参数区别**



- GET：参数在**URL**里，明文、长度有限
- POST：参数在**请求体**里，更安全、支持大数据



2. **POST 也可以 URL 传参**

   比如：`POST /api?name=admin`，这是**URL 参数**，和 body 里的参数不冲突。



3. **后端怎么接收？**



- form 表单：用 `@RequestParam` / `request.getParameter()`
- JSON：用 `@RequestBody` 接收对象
- 文件：用 `@RequestPart` / `MultipartFile`



------

### 总结

1. **普通参数** → form-urlencoded
2. **传文件** → form-data
3. **接口开发** → **JSON（raw）**
4. **纯二进制** → binary