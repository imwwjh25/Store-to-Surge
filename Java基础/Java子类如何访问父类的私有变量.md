在 Java 中，父类的私有成员（使用`private`修饰）被设计为仅能在父类内部访问，子类无法直接访问。但可以通过以下几种方式间接操作，不同方式各有适用场景：

### 一、常用操作方式

#### 1. 父类提供公共的 getter/setter 方法（推荐）

父类通过公共的`getXxx()`和`setXxx()`方法暴露私有成员的读写权限，子类通过调用这些方法间接访问。







```java
// 父类
public class Parent {
    private String privateField; // 私有成员

    // 提供公共的getter方法
    public String getPrivateField() {
        return privateField;
    }

    // 提供公共的setter方法
    public void setPrivateField(String value) {
        this.privateField = value;
    }
}

// 子类
public class Child extends Parent {
    public void accessParentField() {
        // 通过父类的getter/setter访问私有成员
        String value = getPrivateField();
        setPrivateField("new value");
    }
}
```

#### 2. 将私有成员改为受保护的（`protected`）

父类将`private`改为`protected`，允许子类直接访问（但仍限制外部类访问）。









```java
// 父类
public class Parent {
    protected String protectedField; // 受保护成员（子类可访问）
}

// 子类
public class Child extends Parent {
    public void accessParentField() {
        protectedField = "value"; // 子类可直接访问
    }
}
```

#### 3. 通过反射机制强制访问（不推荐）

利用 Java 反射绕过访问权限检查，直接操作父类的私有成员（破坏封装性，不建议在业务代码中使用）。







```java
import java.lang.reflect.Field;

public class Child extends Parent {
    public void accessPrivateField() throws Exception {
        // 获取父类的私有字段
        Field field = Parent.class.getDeclaredField("privateField");
        field.setAccessible(true); // 强制取消访问检查
        
        // 读取私有成员
        Object value = field.get(this);
        // 修改私有成员
        field.set(this, "new value");
    }
}
```

### 二、业务实现中的最佳方案

**推荐使用「父类提供 getter/setter 方法」**，原因如下：



1. **符合封装原则**：
   父类通过方法控制私有成员的访问逻辑（例如添加校验、日志等），避免子类直接修改导致的数据不一致。
   例：父类可以在`setter`中验证参数合法性，防止子类传入无效值。
2. **灵活性和可维护性**：
   若未来父类的私有成员实现逻辑变更（如字段名修改、存储方式变化），只需修改`getter/setter`内部实现，子类无需改动，降低耦合。
3. **安全性**：
   相比`protected`修饰，`private`+`getter/setter`能更精确地控制访问权限（例如只提供`getter`而不提供`setter`，实现 “只读” 效果）。
4. **代码可读性**：
   显式的`getter/setter`方法使代码意图更清晰，其他开发者能快速理解成员的访问规则。

### 三、其他方案的适用场景

- **`protected`修饰**：适用于父类明确允许子类直接操作成员，且无需额外逻辑控制的场景（如工具类中的内部共享变量）。但需注意，`protected`成员在同一包内的其他类也可访问，可能扩大访问范围。
- **反射机制**：仅建议在框架开发、调试工具等特殊场景使用（如序列化 / 反序列化、注解处理器）。业务代码中使用会破坏封装，增加维护成本，且可能因 JVM 安全策略限制而失效。

### 总结

在业务实现中，**优先通过父类的`getter/setter`方法访问私有成员**，这是平衡封装性、灵活性和安全性的最佳实践。只有在明确不需要额外控制逻辑时，才考虑使用`protected`修饰，而反射应尽量避免在业务代码中使用。
