---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java反射API使用方法/","dg-note-properties":{"时间":"2026-03-14"}}
---

#java #最佳实践

```ad-summary
title: 总结

- 反射允许在运行时动态获取类信息、创建对象、调用方法，是框架和工具库的核心机制
- 四个核心类：`Class`、`Field`、`Method`、`Constructor`
- `getXxx()` 只拿 public 含继承，`getDeclaredXxx()` 拿全部含 private 不含继承
- 注解配合反射使用，通过 `isAnnotationPresent` / `getAnnotation` 读取注解信息
- 高频调用要缓存 Method/Field 对象，反射比直接调用慢 10-100 倍
- `setAccessible(true)` 跳过访问检查，框架/工具用，业务代码别乱用
```

## 1. 什么是反射

反射允许在运行时获取程序中每一个类型的成员和成员信息。一般对象的类型在编译期就确定了，而 Java 反射机制可以动态地创建对象并调用其属性，即使这个对象的类型在编译期是未知的。

本质是 JVM 得到 class 对象之后，再通过 class 对象进行反编译，从而获取对象的各种信息。

**优点：** 在运行时获得类的各种内容，进行反编译，能够让我们很方便地创建灵活的代码，这些代码可以在运行时装配，无需在组件之间进行源代码的链接。

**缺点：** 反射会消耗一定的系统资源；反射调用方法时可以忽略权限检查，可能破坏封装性导致安全问题。

**典型应用场景：**
- JDBC 加载驱动：`Class.forName("com.mysql.jdbc.Driver")`
- Spring IOC 容器实例化对象
- 自定义注解生效（反射 + AOP）
- 各类第三方框架的核心实现

## 2. 四个核心类

| 类 | 作用 |
|---|---|
| `Class` | 代表类的实体，表示运行中的类和接口 |
| `Field` | 代表类的成员变量（属性） |
| `Method` | 代表类的方法 |
| `Constructor` | 代表类的构造方法 |

方法命名规律：
- `getXxx(name)`：获取指定 public 成员，包含继承的
- `getXxxs()`：获取所有 public 成员，包含继承的
- `getDeclaredXxx(name)`：获取指定成员（含 private），不含继承的
- `getDeclaredXxxs()`：获取全部成员（含 private），不含继承的

## 3. 获取 Class 对象

```java
// 三种方式，拿到的是同一个对象（JVM 单例）
Class<UserEntity> c1 = UserEntity.class;
Class<?> c2 = Class.forName("com.dd.entity.UserEntity");
Class<?> c3 = new UserEntity().getClass();

System.out.println(c1 == c2); // true
System.out.println(c1 == c3); // true

// 通过 Class 创建实例（调用无参构造）
UserEntity user = c1.newInstance();
```

## 4. 操作字段 Field

```java
// 获取本类所有字段（含 private）
Field[] fields = clazz.getDeclaredFields();

// 读写指定字段
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true); // private 字段必须设置

String name = (String) nameField.get(user);
nameField.set(user, "张三");
```

## 5. 操作方法 Method

```java
// 获取本类所有方法（含 private）
Method[] methods = clazz.getDeclaredMethods();

// 调用 public 方法
Method setName = clazz.getMethod("setName", String.class);
setName.invoke(user, "李四");

// 调用 private 方法
Method privateMethod = clazz.getDeclaredMethod("privateMethod", int.class);
privateMethod.setAccessible(true);
Object result = privateMethod.invoke(user, 100);
```

## 6. 操作构造器 Constructor

```java
// 有参构造
Constructor<UserEntity> c = clazz.getConstructor(String.class, int.class);
UserEntity user = c.newInstance("王五", 25);

// private 无参构造（单例破坏场景）
Constructor<UserEntity> pc = clazz.getDeclaredConstructor();
pc.setAccessible(true);
UserEntity user2 = pc.newInstance();
```

## 7. 注解与反射

注解用来给类声明附加额外信息，可以标注在类、字段、方法等上面。配合反射可以在运行时读取注解信息并做相应处理，这是 Spring、MyBatis 等框架的核心机制之一。

### 元注解

元注解用于声明新注解时指定其特性：

| 元注解 | 作用 |
|---|---|
| `@Target` | 指定注解可标注的位置，如 `ElementType.METHOD`、`ElementType.FIELD` |
| `@Retention` | 指定注解保留到什么阶段，`RetentionPolicy.RUNTIME` 表示运行时可读 |
| `@Inherited` | 标注在父类上时可被子类继承 |

### 自定义注解示例

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value() default "";
}
```

### 反射读取注解

```java
// 扫描带有特定注解的方法并处理
for (Method method : clazz.getDeclaredMethods()) {
    if (method.isAnnotationPresent(MyAnnotation.class)) {
        MyAnnotation ann = method.getAnnotation(MyAnnotation.class);
        System.out.println(method.getName() + " -> " + ann.value());
    }
}
```

## 8. 性能优化：缓存反射对象

反射比直接调用慢 10-100 倍，高频调用场景要缓存 Method/Field 对象：

```java
private static final Map<String, Method> METHOD_CACHE = new ConcurrentHashMap<>();

public Object invoke(Object obj, String methodName, Object... args) {
    String key = obj.getClass().getName() + "#" + methodName;
    Method method = METHOD_CACHE.computeIfAbsent(key, k -> {
        try {
            Method m = obj.getClass().getMethod(methodName);
            m.setAccessible(true);
            return m;
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        }
    });
    try {
        return method.invoke(obj, args);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

## 9. 几个注意点

- `setAccessible(true)` 会跳过访问检查，提升性能，但破坏封装，业务代码里别乱用
- 反射适合框架、工具库、插件系统，不适合普通业务逻辑
- 反射可以越过泛型检查（泛型擦除后运行时无类型约束）
- 性能敏感场景可以考虑 `MethodHandle`（Java 7+），比反射快
