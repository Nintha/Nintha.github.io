---
title: LocalDate求日期间隔
date: 2017-08-02
---
## 前言

    LocalDate类属于java8 Time API系列，那么如何使用LocalDate类来获取两个日期的间隔呢？
    这里主要用了 `LocalDate::toEpochDay()` 方法，这个方法是获取1970-01-01到当前日期所经历的天数。

### 代码示例

``` java
public static long daysBetween(LocalDate from, LocalDate to) {
    long diff = to.toEpochDay() - from.toEpochDay();
    return diff < 0 ? -diff : diff;
}
```

### 运行示例

``` java
public static void main(String[] args) {
    LocalDate from = LocalDate.parse("2017-07-03");
    LocalDate to = LocalDate.parse("2017-07-30");
    long diff = daysBetween(from, to);
    System.out.println("to: " + to.toEpochDay());
    System.out.println("from: " + from.toEpochDay());
    System.out.println("diff: " + diff);
}
```

### 输出结果

```java
to: 17377
from: 17350
diff: 27
```
