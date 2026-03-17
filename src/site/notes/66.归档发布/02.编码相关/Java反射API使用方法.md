---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java反射API使用方法/"}
---

#java #最佳实践

```ad-summary
title: 总结

- 四个核心类：`Class`、`Field`、`Method`、`Constructor`
- `getXxx()` 只拿 public 含继承，`getDeclaredXxx()` 拿全部含 private 不含继承
- 高频调用要缓存 Method/Field 对象，反射比直接调用慢 10-100 倍
- `setAccessible(true)` 跳过访问检查，框架/工具用，业务代码别乱用
```

反射 API 四个核心类：`Class`、`Field`、`Method`、`Constructor`。

方法命名规律：
- `getXxx()`：只获取 public 成员，包含继承的
- `getDeclaredXxx()`：获取所有成员（含 private），不含继承的

## 1. 获取 Class 对象

```java
// 三种方式，拿到的是同一个对象（JVM 单例）
Class<UserEntity> c1 = UserEntity.class;
Class<?> c2 = Class.forName("com.dd.entity.UserEntity");
Class<?> c3 = new UserEntity().getClass();

System.out.println(c1 == c2); // true
```

## 2. 操作字段 Field

```java
// 获取本类所有字段（含 private）
Field[] fields = clazz.getDeclaredFields();

// 读写指定字段
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true); // private 字段必须设置

String name = (String) nameField.get(user);
nameField.set(user, "张三");
```

## 3. 操作方法 Method

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

## 4. 操作构造器 Constructor

```java
// 有参构造
Constructor<UserEntity> c = clazz.getConstructor(String.class, int.class);
UserEntity user = c.newInstance("王五", 25);

// private 无参构造（单例破坏场景）
Constructor<UserEntity> pc = clazz.getDeclaredConstructor();
pc.setAccessible(true);
UserEntity user2 = pc.newInstance();
```

## 5. 结合注解使用

```java
// 扫描带有特定注解的方法并处理
for (Method method : clazz.getDeclaredMethods()) {
    if (method.isAnnotationPresent(MyAnnotation.class)) {
        MyAnnotation ann = method.getAnnotation(MyAnnotation.class);
        System.out.println(method.getName() + " -> " + ann.value());
    }
}
```

## 6. 性能优化：缓存反射对象

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

## 7. 几个注意点

- `setAccessible(true)` 会跳过访问检查，提升性能，但破坏封装，业务代码里别乱用
- 反射适合框架、工具库、插件系统，不适合普通业务逻辑
- 性能敏感场景可以考虑 `MethodHandle`（Java 7+），比反射快

