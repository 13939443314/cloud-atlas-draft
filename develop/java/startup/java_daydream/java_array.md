# 创建数组

* 创建数组并赋值

```java
int[] smallPrimes = {2, 3, 5, 7, 11, 13};
```

> 创建数组时不需要调用`new`

* 创建匿名数组

```java
new int[] {17, 19, 23, 29, 31, 37};
```

* 可以在不创建新的数组变量情况下重新初始化一个数组

```java
smallPrimes = new int[] {17, 19, 23, 29, 31, 37};
```

> 注意：重新初始化数组值需要确保前后两次赋值的数值个数一一对应！！！

或者写成:

```java
int[] anonymous = {17, 19, 23, 29, 31, 37};
smallPrimes = anonymous;
```

# 数组拷贝

在java中，如果使用 `=` 来"copy"数组，则实际上就是两个变量同时引用同一个数组（相当于两个软链接连接到同一个数组变量上）。所以，此时修改一个数组的值会同时修改另一个数组的值。

如果要完全分离两个数组，实现把一个数组的所有值赋值到一个新的数组中，则需要使用 `Arrays` 类的 `copyOf` 方法，这种方法通常用来增加数组的大小：

```java
luckyNumbers = Arrays.copyOf(luckyNumbers, 2 * luckNumbers.length);
```

如果数组元素是数值型，则多余的元素被赋值为0；如果是布尔型则多余的元素赋值为false。相反，如果长度小雨原始数组，则只赋值最前面的元素。