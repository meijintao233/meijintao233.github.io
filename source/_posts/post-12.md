---
title: "[译]Javascript 中你需要知道的一切关于日期的事情"
date: 2019-09-15 19:50:45
tags:
  - 翻译
  - 前端
  - frontend
---

> 原文地址：[Everything You Need to Know About Date in JavaScript](https://css-tricks.com/everything-you-need-to-know-about-date-in-javascript/?source=post_page---------------------------)
> 原文作者：[Zell Liew](https://zellwk.com/blog/)
> 译者: [charlie mei](https://meijintao233.github.io/)

在 Javascript 中日期很奇怪，它让我们紧张不安，以至于我们需要找到其他库(像 Date-fns 和 Moment)在我们需要使用日期和时间的时候(哈！)。

但是我们不总是需要使用库。如果你知道需要注意些什么，日期实际上相当简单。在本文中，我会引导你在 Javascript 中你需要知道的一切关于日期的事情。

首先，我们先承认时区的存在。

---

## 时区

世界上有数以百计的时区。在 Javascript 中，我们只关心两个——本地时间和协调世界时(UTC)。

- **本地时间**涉及你电脑所在的时区
- **UTC**是现实中格林尼治标准时间(GMT)的代名词

在 Javascript 中(除了一个)几乎每个日期方法默认都会给你本地时间或者日期。假如你指定了 UTC 那么你只能得到 UTC。

有了这个，我们可以谈一下创建日期了。

---

## 创建日期

你可以用`new Date()`创建日期。有四种可能的方式使用`new Date()`：

- 日期字符串
- 日期参数
- 时间戳
- 没有参数

---

 <!-- more -->

## 日期字符串方法

在日期字符串方法中，通过给`new Date()`传入日期字符串，你可以创建日期。

```javascript
new Date("1988-03-21");
```

在写日期的时候，我们倾向于使用日期字符串。这很自然因为我们在生活中一直使用日期字符串。

如果我写下`21-03-1988`， 你推导出是 1988 年 3 月 21 日说没有问题的。是吗？但是如果你在 Javascript 里写`21-03-1988`，你会得到一个`Invalid Date`。

![img](https://css-tricks.com/wp-content/uploads/2019/06/new-date-invalid.png)

对于这个有充分的理由：

在世界的不同地方，我们翻译日期字符串的方式不同。例如`11-06-2019`不是`2019年11月6日`就是`2019年6月11日`。但是你不能确定我指的是哪一个，除非你指定我用的日期系统。

在 Javascript 中，如果你想使用日期字符串，你需要使用一个全世界都接受的格式。其中一个格式就是 ISO 8601 扩展格式。

```javascript
// ISO 8601 Extended format
`YYYY-MM-DDTHH:mm:ss:sssZ`;
```

这是值所表示的意思：

- `YYYY`：4 位数字年份
- `MM`：2 位数字月份(其中一月是 01，12 月是 12)
- `DD`：2 位数字日期(0 到 31)
- `-`： 日期分隔符
- `T`：表明时间开始
- `HH`：24 位数字小时(0 到 23)
- `mm`：分(0 到 59)
- `ss`：秒(0 到 59)
- `sss`：毫秒(0 到 999)
- `:`：时间分隔符
- `Z`：如果`Z`存在，日期会被设置成 UTC。如果`Z`不存在，它会是本地时间(这仅适用于提供时间的情况)

如果你创建日期，小时，分，秒和毫秒是可选的。那么，如果你想创建 2019 年 6 月 11 日的时间，你可以这样写：

```javascript
new Date("2019-06-11");
```

在这里要特别注意。使用日期字符串创建日期会一个巨大的问题。如果你`console.log`这个日期你会发现问题。

如果你生活的区域在 GMT 之后，你会得到一个 6 月 10 日的日期。

![img](https://css-tricks.com/wp-content/uploads/2019/06/datestring-10june.png)

如果你生活的区域在 GMT 之前，你会得到一个 6 月 11 日的日期。

发生这个情况是因为日期字符串有一个奇怪的行为：如果你创建日期的时候没有指定时间，你得到的日期会被设置成 UTC。

在上面的脚本中，当你写入`new Date("2019-06-11")`，实际上你创建了一个`11th June, 2019, 12am UTC`日期。这是生活在 GMT 之前的人会得到`10th June`而不是`11th June`的原因。

如果你想用日期字符串创建当地时间的日期，你需要包括时间。当你包括时间的时候，你需要写入`HH`和`mm`的最小值(或者谷歌浏览器返回一个无效日期)。

```javascript
new Date("2019-06-11T00:00");
```

![img](https://css-tricks.com/wp-content/uploads/2019/06/datestring-local.png)

和日期字符串有关的当地时间和 UTC 是一个可能的错误来源，这很难捕获。所以，我推荐你不要用日期字符串创建日期。

(顺便提一下，MDN 警告反对日期字符串方法，因为浏览器可以不同地解析日期字符串)

![img](https://css-tricks.com/wp-content/uploads/2019/06/mdn.png)

如果你想创建日期，使用参数或者时间戳。

---

## 用参数创建日期

你可以传入七个参数去创建日期/时间。

- Year：四位数字年份。
- Month：一年中的月份(0-11)。月份从 0 开始索引。如果省略默认为 0。
- Day：一月中的天数(1-31)。如果省略默认为 1.
- Hour：一天中的小时数(0-59)。如果省略默认为 0
- Minutes：分钟(0-59)。如果省略默认为 0。
- Seconds：秒(0-59)。如果省略默认为 0。
- Milliseconds：毫秒(0-999)。如果省略默认为 0。

```javascript
// 11th June 2019, 5:23:59am, Local Time
new Date(2019, 5, 11, 5, 23, 59);
```

许多开发者(包括我自己)都避免使用参数方法，因为它看起来是复杂的。但实际上是相当简单。

尝试从左往右读数。从左到右你插入的值越来越小：年，月，日，时，分，秒，毫秒。

```javascript
new Date(2017, 3, 22, 5, 23, 50);

// This date can be easily read if you follow the left-right formula.
// Year: 2017,
// Month: April (because month is zero-indexed)
// Date: 22
// Hours: 05
// Minutes: 23
// Seconds: 50
```

与日期有关的问题最多的部分是月份值是从 0 开始索引的，就像`January === 0`，`February === 1`，`March === 2`等等。

我们不知道为什么 Javascript 中月份从 0 开始索引，但是与其争论为什么一月份应该是 1(不是 0)，更好的是接受在 Javascript 中月份是从 0 开始索引。一旦你接受了这个事实，使用日期会变得相当简单。

下面是一些例子，会让你熟悉：

```javascript
// 21st March 1988, 12am, Local Time.
new Date(1988, 2, 21);

// 25th December 2019, 8am, Local Time.
new Date(2019, 11, 25, 8);

// 6th November 2023, 2:20am, Local Time
new Date(2023, 10, 6, 2, 20);

// 11th June 2019, 5:23:59am, Local Time
new Date(2019, 5, 11, 5, 23, 59);
```

用参数创建的日期都是本地时间吗？

那是用参数的好处之一——你不会在本地时间和 UTC 之间感到困惑。如果你真的需要 UTC，你可以用这个方式创建 UTC 日期。

```
// 11th June 2019, 12am, UTC.
new Date(Date.UTC(2019, 5, 11))
```

## 用时间戳创建日期

在 Javascript 中，时间戳是从 1970 年 1 月经过的毫秒数(1970 年 1 月也被称为 Unix 时间)。根据我的经验，你们很少使用时间戳创建时间。你们仅用时间戳在不同时间中进行比较(更多相关的在后面)。

```javascript
// 11th June 2019, 8am (in my Local Time, Singapore)
new Date(1560211200000);
```

## 不用参数创建日期

如果你不用参数创建日期，你会得到一个被设置成当前时间的日期(本地时间)。

```javascript
new Date();
```

![img](https://css-tricks.com/wp-content/uploads/2019/06/now.png)

你可以从图像中看出是我写这篇文章的时间是新加坡的 5 月 25 日早上 11 时 19 分。

## 创建时间的总结

- 可以用`new Date()`创建时间
- 有四种可能的语法：
  - 日期字符串
  - 参数
  - 时间戳
  - 无参
- 永远不要用日期字符串方法创建日期。
- 最好用传参方法创建日期。
- 记住(并接受)Javascript 中月份是从 0 开始索引的。

接下来，我们讨论一下将日期转换成可读的字符串。

## 格式化日期

大多数编程语言为你提供一个格式化工具去创建任意你要的日期格式。例如，在 PHP 里，你可以写`date("d M Y")`把日期转换成`23 Jan 2019`。

但是 Javascript 中没有容易的方式格式化日期。

原生日期对象附带七个格式化方法。每一个方法会给你提供一个特定的值(相当的没用)。

```javascript
const date = new Date(2019, 0, 23, 17, 23, 42);
```

- `toString`返回`Wed Jan 23 2019 17:23:42 GMT+0800 (Singapore Standard Time)`。
- `toDateString`返回`Wed Jan 23 2019`。
- `toLocaleString`返回`23/01/2019, 17:23:42`。
- `toLocaleDateString`返回`23/01/2019`。
- `toGMTString`返回`Wed, 23 Jan 2019 09:23:42 GMT`
- `toUTCString`返回`Wed, 23 Jan 2019 09:23:42 GMT`
- `toISOString`返回`2019-01-23T09:23:42.079Z`

如果你需要自定义格式，需要自己创建。

## 实现自定义日期格式

假设你想要像`Thu, 23 January 2019`的格式。为了创建这个值，你需要知道(和使用)日期对象附带的日期方法。

你可以用这四个方法得到日期：

- `getFullYear`：根据本地时间得到四位数字年份
- `getMonth`：根据本地时间得到一年中的月份(0-11)。月份从 0 开始索引
- `getDate`：根据本地时间得到一月中的日期(1-31)。
- `getDay`：根据本地时间得到一周中的星期几(0-6)。一周从周日(0)开始，周六(6)结束。

为了`Thu, 23 January 2019`创建`23`和`2019`很简单，我们可以使用`getFullYear`和`getDate`去获取。

```javascript
const d = new Date(2019, 0, 23);
const year = d.getFullYear(); // 2019
const date = d.getDate(); // 23
```

获取`Thu`和`January`困难。

为了获取`January`，你可以创建一个映射十二个月份到它们各自名字的对象。

```javascript
const months = {
  0: "January",
  1: "February",
  2: "March",
  3: "April",
  4: "May",
  5: "June",
  6: "July",
  7: "August",
  8: "September",
  9: "October",
  10: "November",
  11: "December"
};
```

既然月份是从 0 开始索引的，那么我们可以用数组而不是对象。它会产生同样的结果。

```javascript
const months = [
  "January",
  "February",
  "March",
  "April",
  "May",
  "June",
  "July",
  "August",
  "September",
  "October",
  "November",
  "December"
];
```

为了获取`January`，我们需要：

- 用`getMonth`从日期中得到从 0 开始索引的月份
- 从`months`中获取月份名字

```javascript
const monthIndex = d.getMonth();
const monthName = months[monthIndex];
console.log(monthName); // January
```

精简版：

```javascript
const monthName = months(d.getMonth());
console.log(monthName); // January
```

你可以以同样的方式获取`Thu`。这次，我们需要一个包含一周七天的数组。

```javascript
const days = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
```

然后你：

- 用`getDay`获取`dayIndex`
- 用`dayIndex`去获取`dayName`

```javascript
const dayIndex = d.getDay();
const dayName = days[dayIndex]; // Thu
```

精简版：

```javascript
const dayName = days[d.getDay()]; // Thu
```

然后，你把你创建的变量组合起来，就可以得到格式化了的字符串。

```javascript
const formatted = `${dayName}, ${date} ${monthName} ${year}`;
console.log(formatted); // Thu, 23 January 2019
```

是的，它是乏味的。但是一旦掌握它，这并非不可能。

如果你曾经需要创建一个自定义的格式化时间，你可以使用以下方法：

- `getHours`：根据本地时间获取小时(0-23)
- `getMinutes`：根据本地时间获取分钟(0-59)
- `getSeconds`：根据本地时间获取秒(0-59)
- `getMilliseconds`：根据本地时间获取毫秒(0-999)

下一步，我们讨论一下日期比较。

## 日期比较

如果你想知道一个日期是否在另一个之前或者之后，你可以直接用`>`，`<`，`>=`和`<=`对它们进行比较。

```javasc
const earlier = new Date(2019, 0, 26)
const later = new Date(2019, 0, 27)

console.log(earlier < later) // true
```

更困难的是如果你想检测两个日期是否是完全相同的时间。你不能用`==`或者`===`进行比较。

```javascript
const a = new Date(2019, 0, 26);
const b = new Date(2019, 0, 26);

console.log(a == b); // false
console.log(a === b); // false
```

为了检测两个日期是否是相同的时间，你可以用`getTime`检测它们的时间戳。

```javascript
const isSameTime = (a, b) => {
  return a.getTime() === b.getTime();
};

const a = new Date(2019, 0, 26);
const b = new Date(2019, 0, 26);
console.log(isSameTime(a, b)); // true
```

如果你想检测两个日期是否是相同的天，你可以检测它们`getFullYear`，`getMonth`和`getDate`返回的值。

```javascript
const isSameDay = (a, b) => {
  return (
    a.getFullYear() === b.getFullYear() &&
    a.getMonth() === b.getMonth() &&
    a.getDate() === b.getDate()
  );
};

const a = new Date(2019, 0, 26, 10); // 26 Jan 2019, 10am
const b = new Date(2019, 0, 26, 12); // 26 Jan 2019, 12pm
console.log(isSameDay(a, b)); // true
```

还有最后一件事我们得涵盖的。

## 从另一个日期获取日期

有两个可能的你想从另一个日期获取日期的脚本。

- 从另一个日期设置特定的日期/时间值
- 从另一个日期增/减一个增量

## 设置一个特定的日期/时间

你可以使用这些方法从另一个日期设置日期/时间：

- `setFullYear`：在本地时间设置 4 位数年份
- `setMonth`：在本地时间设置月份
- `setDate`：在本地时间设置天数
- `setHours`：在本地时间设置小时
- `setMinutes`：在本地时间设置分钟
- `setSeconds`：在本地时间设置秒
- `setMilliseconds`：在本地时间设置毫秒

例如，如果你想把日期设置成本月中的第 15 天，你可以用`setDate(15)`。

```javascript
const d = new Date(2019, 0, 10);
d.setDate(15);

console.log(d); // 15 January 2019
```

如果你想把月份设置成六月，你可以用`setMonth`。(记住，Javascript 中月份是从 0 开始索引的！)

```javascript
const d = new Date(2019, 0, 10);
d.setMonth(5);

console.log(d); // 10 June 2019
```

注意：上面的 setter 方法会改变原始时间对象。实践中，我们不应该改变对象。相反我们应该在新的时间对象上执行这些操作。

```javascript
const d = new Date(2019, 0, 10);
const newDate = new Date(d);
newDate.setMonth(5);

console.log(d); // 10 January 2019
console.log(newDate); // 10 June 2019
```

## 从另一个日期增/减一个增量

增量是一个变化。从另一个日期加/减一个增量，我意思是：你想从另一个日期获取一个 X 日期。可能是 X 年，X 月，x 日等等。

为了获取增量，你需要知道当前日期的值。你可以用这些方法获取它：

- `getFullYear`：根据本地时间得到四位数字年份
- `getMonth`：根据本地时间得到一年中的月份(0-11)。月份从 0 开始索引
- `getDate`：根据本地时间得到一月中的日期(1-31)。
- `getDay`：根据本地时间得到一周中的星期几(0-6)。一周从周日(0)开始，周六(6)结束。
- `getHours`：根据本地时间获取小时(0-23)
- `getMinutes`：根据本地时间获取分钟(0-59)
- `getSeconds`：根据本地时间获取秒(0-59)
- `getMilliseconds`：根据本地时间获取毫秒(0-999)

有两个常见方法增/减一个增量。第一个在 Stack Overflow 更受欢迎。它简洁，但很难掌握。第二个方法更加冗长，但很容易理解。

我们来看看这两种方法。

假设你想获得一个三天后的日期。在这个例子中，我们也假定今天是 2019 年 3 月 28 日。(当我们使用固定日期时这更容易解释)。

```javascript
// Assumes today is 28 March 2019
const today = new Date(2019, 2, 28);
```

首先，我们创建一个新的日期对象(这样我们没有改变原始日期对象)

```javascript
const finalDate = new Date(today);
```

接着，我们需要知道我们想改变的值。既然我们打算改变天数，我们可以用`getDate`获取天。

```javascript
const currentDate = today.getDate();
```

我们想要一个三天后的日期。我们将增加增量(3)到当前日期。

```javascript
finalDate.setDate(currentDate + 3);
```

设置方式的完整的代码：

```javascript
const today = new Date(2019, 2, 28);
const finalDate = new Date(today);
finalDate.setDate(today.getDate() + 3);

console.log(finalDate); // 31 March 2019
```

现在我们使用`getFullYear`，`getMonth`，`getDate`和其他 getter 方法直到我们达到我们想要改变的值的类型。然后，我们用`new Date()`创建最终日期。

```javascript
const today = new Date(2019, 2, 28);

// Getting required values
const year = today.getFullYear();
const month = today.getMonh();
const day = today.getDate();

// Creating a new Date (with the delta)
const finalDate = new Date(year, month, day + 3);

console.log(finalDate); // 31 March 2019
```

两种方法都有效。选一种并坚持下去。

## 自动日期校正

如果你提供一个超过它可接受范围的值给 Date，Javascript 会自动为你重新计算日期。

这是一个例子。让我们假设我们设置日期为 2019 年 3 月 33 日。(日历上没有 3 月 33 日)。这种情况下，Javascript 自动调整 3 月 33 日到 4 月 2 日。

```javasc
// 33rd March => 2nd April
new Date(2019, 2, 33)
```

![img](https://css-tricks.com/wp-content/uploads/2019/06/33march.png)

这意味着你不需要担心计算分钟，小时，天，月等等。当创建一个增量，Javascript 自动为你处理它。

```javasc
// 33rd March => 2nd April
new Date(2019, 2, 30 + 3)
```

![img](https://css-tricks.com/wp-content/uploads/2019/06/33march-2.png)

以上就是你需要知道的关于 Javascript 原生日期对象的所有东西。
