---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java Stream API常用操作方法/","dg-note-properties":{"时间":"2026-03-14"}}
---

#java

```ad-summary
title: 总结

- Stream 操作分两类：中间操作（惰性，返回 Stream）和终止操作（触发计算，返回结果）
- 常用：filter 过滤、map 映射、groupingBy 分组、sorted 排序、distinct 去重
- Stream 不能复用；数据量小时和 for 循环性能差不多，别过度优化
```

Stream 操作分两类：
- **中间操作**（返回 Stream）：filter、map、sorted 等，惰性执行
- **终止操作**（返回结果）：collect、count、reduce 等，触发实际计算

## 1. 过滤 filter
```java
// 单条件
List<User> failed = list.stream()
    .filter(u -> u.getScore() < 60)
    .collect(Collectors.toList());

// 多条件
List<User> result = list.stream()
    .filter(u -> u.getScore() >= 60 && u.getGrade() == 1)
    .collect(Collectors.toList());
```

---

## 2. 映射 map

```java
// 提取字段列表
List<String> names = list.stream()
    .map(User::getName)
    .collect(Collectors.toList());

List<Integer> ids = list.stream()
    .map(User::getId)
    .collect(Collectors.toList());
```

---

## 3. 排序 sorted

```java
// 升序
list.stream().sorted(Comparator.comparing(User::getId));

// 降序
list.stream().sorted(Comparator.comparing(User::getId).reversed());
```

---

## 4. 去重 distinct

```java
// 基于 equals() 去重，记得重写 equals 和 hashCode
list.stream().distinct().collect(Collectors.toList());
```

---

## 5. 统计

```java
// 求和
double sum = list.stream().mapToDouble(User::getScore).sum();

// 平均值
double avg = list.stream().mapToDouble(User::getScore).average().orElse(0.0);

// 计数
long count = list.stream().filter(u -> u.getScore() >= 60).count();
```

---

## 6. 分组 groupingBy

```java
// 按年级分组
Map<Integer, List<User>> byGrade = list.stream()
    .collect(Collectors.groupingBy(User::getGrade));

// 多重分组：年级 → 班级
Map<Integer, Map<Integer, List<User>>> byGradeAndClass = list.stream()
    .collect(Collectors.groupingBy(
        User::getGrade,
        Collectors.groupingBy(User::getClasses)
    ));

// 分组统计每个年级人数
Map<Integer, Long> countByGrade = list.stream()
    .collect(Collectors.groupingBy(User::getGrade, Collectors.counting()));

// 分组统计每个年级总分
Map<Integer, Double> sumByGrade = list.stream()
    .collect(Collectors.groupingBy(
        User::getGrade,
        Collectors.summingDouble(User::getScore)
    ));
```

---

## 7. 常用 Collectors

```java
Collectors.toList()                              // → List
Collectors.toSet()                               // → Set
Collectors.toMap(User::getId, User::getName)     // → Map
Collectors.joining(", ")                         // 拼接字符串
Collectors.partitioningBy(u -> u.getScore() >= 60) // 按 boolean 分两组
Collectors.maxBy(Comparator.comparing(User::getScore)) // 最大值
```

---

## 8. 链式组合示例

```java
// 每个年级及格人数
Map<Integer, Long> passCount = list.stream()
    .filter(u -> u.getScore() >= 60)
    .collect(Collectors.groupingBy(User::getGrade, Collectors.counting()));

// 每个年级分数最高的学生
Map<Integer, Optional<User>> topByGrade = list.stream()
    .collect(Collectors.groupingBy(
        User::getGrade,
        Collectors.maxBy(Comparator.comparing(User::getScore))
    ));
```

---
## 9. 几个注意点
- Stream 用完就关闭，不能复用，需要重新从集合创建
- 数据量小时 Stream 和 for 循环性能差不多，别过度优化
- `parallelStream()` 可以多核并行，但要注意线程安全，不是所有场景都适合
