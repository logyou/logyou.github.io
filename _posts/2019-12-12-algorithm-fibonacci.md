---
title: 斐波那契数列
description: 尝试使用不同的方法，分别对比执行效率。
---

# 一、递归

```java
private static long f1(int n) {
    if (n < 1) {
        return 0;
    } else if (n < 2) {
        return 1;
    } else {
        return f1(n - 1) + f1(n - 2);
    }
}
```
# 二、缓存

```java
private static long f2(int n) {
    HashMap<Integer, Long> map = new HashMap<>();
    map.put(0, 0L);
    map.put(1, 1L);
    return memo(map, n);
}

private static long memo(HashMap<Integer, Long> map, int n) {
    if (!map.containsKey(n)) {
        map.put(n, memo(map, n - 1) + memo(map, n - 2));
    }
    return map.get(n);
}
```
# 三、动态

```java
private static long f3(int n) {
    if (n < 2) {
        return n;
    }
    long prev = 0, curr = 1;
    for (int i = 0; i < n - 1; i++) {
        long sum = prev + curr;
        prev = curr;
        curr = sum;
    }
    return curr;
}
```
# 对比结果
## n=20

执行方法 | 计算结果 |循环次数 | 时间复杂度
---|---|---|---
f1 | 6765 | 21891 | O(2^n^)
f2 | 6765 | 39 | O(n)
f3 | 6765 | 29 | O(1)

## n=50

执行方法 | 计算结果 |循环次数 | 时间复杂度
---|---|---|---
f1 | 12586269025 | 40730022147 | O(2^n^)
f2 | 12586269025 | 99 | O(n)
f3 | 12586269025 | 49 | O(1)

从以上结果分析可以看出，递归效率最低，循环次数呈指数增长。
