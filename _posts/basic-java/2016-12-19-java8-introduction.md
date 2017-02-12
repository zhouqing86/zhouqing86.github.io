---
layout: post
comments: false
categories: Java基础
date:   2016-12-19 18:30:54
title: Java8介绍
---

<div id="toc"></div>

## 新特性

### Lambda

Lambda表达式的语法是`(argument) -> (body)`：

```
Runnable r1 = () -> System.out.println("My Runnable");
```

如何来理解上面的语句呢：

- Runnable是一个函数式接口，注解为@FunctionalInterface

	```
	@FunctionalInterface
	public interface Runnable {
	  public abstract void run();
	}
	```

	> Java8中增加了@FunctionalInterface注解，可以将一个接口注解为函数接口。@FunctionalInterface注解的接口只能有一个抽象方法，如果多余一个抽象方法编译时会报错。

- Runnable接口中的run方法没有参数，所以lambda表达式中也没有参数

- 在body中只有一个语句，所以可以省略掉{}

Lambda表达式意味着可以创建匿名类，但是其并没有带来任何运行时的好处，那我们为什么要用它呢？

- 比起匿名类来说，其减少代码行数

- 融入Stream API的序列化和支持并发操作

	举个例子来说明序列化的操作，如判断某个数是否为素数，Java8之前的传统写法可能是：

	```
	private static boolean isPrime(int number) {
		if(number < 2) return false;
		for(int i=2; i<number; i++){
			if(number % i == 0) return false;
		}
		return true;
	}
	```

	使用Lambda表达式和Stream后

	```
	private static boolean isPrime(int number) {		
		return number > 1
				&& IntStream.range(2, number).noneMatch(
						index -> number % index == 0);
	}
	```

	noneMatch中的lambda表达式可以提取出来变成一个接口局部变量

	```
	private static boolean isPrime(int number) {
		IntPredicate isDivisible = index -> number % index == 0;

		return number > 1
				&& IntStream.range(2, number).noneMatch(isDivisible);
	}
	```

	> Java8的很多集合API被重写，引入了Stream API，其使用了很多函数式接口。Java8已经在java.util.function中定义了很多函数式接口，如Consumer, Supplier, Function和Predicate。

- 函数可以当做参数来传递

- 延迟计算

	```
	public static int findSquareOfMaxOdd(List<Integer> numbers) {
		return numbers.stream()
				.filter(NumberTest::isOdd) 		//Predicate is functional interface and
				.filter(NumberTest::isGreaterThan3)	// we are using lambdas to initialize it
				.filter(NumberTest::isLessThan11)	// rather than anonymous inner classes
				.max(Comparator.naturalOrder())
				.map(i -> i * i)
				.get();
	}

	public static boolean isOdd(int i) {
		return i % 2 != 0;
	}

	public static boolean isGreaterThan3(int i){
		return i > 3;
	}

	public static boolean isLessThan11(int i){
		return i < 11;
	}
	```

	> :: 是Java8引入的语法糖，`NumberTest::isGreaterThan3`是`i -> NumberTest.isGreaterThan3(i)`的简写。


### Stream API
Java8中新增了java.util.stream，并在Collection接口中增加了对Stream的支持:

```
default Stream<E> stream() {
  return StreamSupport.stream(spliterator(), false);
}

default Stream<E> parallelStream() {
  return StreamSupport.stream(spliterator(), true);
}
```

为什么要引入Stream呢，我们来看一个例子，计算大于10的所有整数的和：

```
private static int sumIterator(List<Integer> list) {
	Iterator<Integer> it = list.iterator();
	int sum = 0;
	while (it.hasNext()) {
		int num = it.next();
		if (num > 10) {
			sum += num;
		}
	}
	return sum;
}
```

上面代码存在的问题是:

- 我们仅仅想知道整数的和，但是我们必须知道list是怎么迭代的

- 程序是连续的，我们很难再程序中做到并发执行

- 简单的任务也需要些很多代码

Stream的引入就是为了解决上面代码存在的问题。

```
private static int sumStream(List<Integer> list) {
	return list.stream().filter(i -> i > 10).mapToInt(i -> i).sum();
}
```

Java Stream不存储数据，其在collection和array等数据结构上操作并提供管道数据，以便我们在数据上进行一些特定的操作，如filter, map等。为了提供集合对int, long的支持且避免自动拆装箱，一些特定的Steam被设计出来， IntStream, LongStream, DoubleStream。

> 如何创建Stream呢

- `Stream<Integer> stream = Stream.of(1,2,3,4);`

- `Stream<Integer> stream = Stream.of(new Integer[]{1,2,3,4}); `, 但是注意不能是`new int[]`，因为其不支持自动装箱

- `myList.stream()`或`myList.parallelStream()`

- `Stream.generate(() -> {return "abc";});`或`Stream.iterate("abc", (i) -> i);`

- `LongStream is = Arrays.stream(new long[]{1,2,3,4});`或`IntStream is2 = "abc".chars();`

> 如何从Stream中得到Collection或Array呢

- `List<Integer> intList = intStream.collect(Collectors.toList());`

- `Integer[] intArray = intStream.toArray(Integer[]::new);`

Java Stream API的一些限制:

- 需要使用没有状态的lambda表达式，有状态的lambda表达式会导致结果不可预测

- 一旦创建的stream被消费，后续就不能再被用

- Stream API中的大量方法需要学习

### Optional
Optional类是一个可以为null的容器对象。如果值存在则isPresent()方法会返回true，调用get()方法会返回该对象。

```
Optional<String> name = Optional.of("Sanaulla");
System.out.println(name.get());
name.ifPresent((value) -> {
  System.out.println("The length of the value is: " + value.length());
});

```

### forEach

Java8在Iterable接口中引入forEach方法，`myList.forEach(t -> System.out.println(t));`。

forEach方法接受`java.util.function.Consumer`作为其参数。

### 接口中的default和static方法

Java8中接口中可以有实现代码，可以使用default或static关键字来修饰有具体实现的方法。

### Java Time API

- `LocalDate today = LocalDate.now();`

- `LocalDate firstDay_2014 = LocalDate.of(2014, Month.JANUARY, 1);`

- `LocalDate hundredDay2014 = LocalDate.ofYearDay(2014, 100);`

- `LocalTime specificTime = LocalTime.of(12,20,25,40);`

- `LocalDateTime today = LocalDateTime.now();`

- `Instant timestamp = Instant.now();`

- `today.with(TemporalAdjusters.firstDayOfMonth())`

- `today.format(DateTimeFormatter.ofPattern("d::MMM::uuuu"))`

### jdeps命令

jdeps是一个相当棒的命令行工具，它可以展示包层级和类层级的Java类依赖关系，它以.class文件、目录或者Jar文件为输入，然后会把依赖关系输出到控制台。 如`jdeps org.springframework.core-3.0.5.RELEASE.jar`。

## 加强特性

### Collection加强

- Iterator中增加了forEachRemaining(Consumer action)方法

- Collection中removeIf(Predicate filter)， Collection spliterator()

- Map中replaceAll(), compute(), merge()

### Concurrency加强

- ConcurrentHashMap compute(), forEach(), forEachEntry(), forEachKey(), forEachValue(), merge(), reduce() 和 search()方法

- CompletableFuture

- Executors newWorkStealingPool()方法

### Java IO加强

- Files.list(Path dir)读取一个目录，返回一个所有目录中条目的stream

- Files.lines(Path path)读取整个文件的所有行返回stream

- Files.find()

- BufferedReader.lines()

## 参考资料
- [Java 8 Features with Examples](http://www.journaldev.com/2389/java-8-features-with-examples)
- [Java 8 Functional Interfaces](http://www.journaldev.com/2763/java-8-functional-interfaces)
- [Java 8 Stream](http://www.journaldev.com/2774/java-8-stream)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
