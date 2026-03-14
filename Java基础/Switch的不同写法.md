# 1. 新版 Switch（Java 14+ / SpringBoot 默认用这个）

只用 `->`，**不用 break，不用冒号**，清爽！





```
public class Test {
    public static void main(String[] args) {
        String type = "ADD";

        // ✅ 新版 Switch（Java 14+）
        switch (type) {
            case "ADD" -> System.out.println("执行新增");
            case "UPDATE" -> System.out.println("执行修改");
            default -> System.out.println("未知类型");
        }
    }
}
```

## 新版特点：

- 用 **`->`**，**绝对不用 `:`**
- 不用写 `break`（不会穿透）
- 代码更短更安全

------

# 2. 旧版 Switch（你混掉的那种）










```
switch (type) {
    case "ADD" : 
        System.out.println("执行新增");
        break;
    case "UPDATE" : 
        System.out.println("执行修改");
        break;
    default : 
        System.out.println("未知类型");
}
```

------

# 3. 你问的重点：Java 17 后不用判断非空？

对！**Java 17 增强版 Switch 会自动报空指针，不用你手动判空**

以前你要写：










```
if(str == null) return;
```

现在 **Java 17+** 直接写：







```
switch (str) {
    case "A" -> ...
}
```

如果 str = null

**会直接抛出 NullPointerException，明确告诉你是空**

所以不用自己写 `if(str != null)` 了！

------

# 4. 最干净的最终版（你直接复制用）








```
public class Test {
    public static void main(String[] args) {
        String opt = "DELETE";

        // ✅ 标准新版 Switch，无冒号、无break、不混淆
        switch (opt) {
            case "INSERT" -> System.out.println("新增");
            case "DELETE" -> System.out.println("删除");
            default -> System.out.println("其他");
        }
    }
}
```

------

## 一句话记住不混淆

- **新版：`->` 不用冒号，不用 break**
- **旧版：`:` 必须加 break**
- **Java 17+：switch 里不用判断 null，框架自动处理**