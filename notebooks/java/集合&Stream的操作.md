# 集合、Stream的操作

[toc]



分组：

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4);
//分组
Map<Integer, List<Integer>> group = list.stream().collect(Collectors.groupingBy(i -> i % 2));
```



## Stream流运行机制

首先分为 **中间操作** 和 **最终操作**，**<u>在最终操作没有调用的情况下，所有的中级操作都不会执行</u>**。

最终操作里面分为**短路操作**和**非短路操作**，短路操作就是limit/findxxx/xxxMatch这种，就是找了符合条件的就终止，其他的就是非短路操作。在无限流里面需要调用短路操作，否则像炫迈口香糖一样根本停不下来！

中间操作又分为 **有状态操作**和**无状态操作**，状态就是和其他数据有关系。如map/filter方法，只有一个参数就是自己，就是无状态操作；而distinct/sorted就是有状态操作，因为去重和排序都需要和其他数据比较。

### 运行机制

**对于有状态操作，无法使用并行。**

**Stream会将有状态操作前后的 无状态操作合并执行，即对于一系列的无状态操作，会在同一个线程中一起执行。（看例子2）**

NOTE: 

所以最佳实践是，**将无状态放一起，再执行有状态操作**。

*例子1：*

```java
	@Test
	public void test_parallel() {
		List<String> collect = Stream.of("one", "two", "three", "four")
				.filter(e -> e.length() > 3)
				.peek(e -> System.out.println(Thread.currentThread().getName() + "-- Filtered value: " + e))
				.map(String::toUpperCase)
				.peek(e -> System.out.println(Thread.currentThread().getName() + "-- Mapped value: " + e))
				.parallel()
				.collect(Collectors.toList());
		System.out.println(collect);
	}
```

输出：

```shell
main-- Filtered value: three
ForkJoinPool.commonPool-worker-5-- Filtered value: four
main-- Mapped value: THREE
ForkJoinPool.commonPool-worker-5-- Mapped value: FOUR
[THREE, FOUR]
```



*例子2：*

```java
	@Test
	public void test_parallel() {
		Random random = new Random(100000);
		Stream.generate(random::nextInt)
				.limit(10)
				.peek(i -> {
					System.out.println(Thread.currentThread().getName() + " peek:" + i);
				})
				.filter(i -> {
					System.out.println(Thread.currentThread().getName() + " filter:" + i);
					return i > 0;
				})
				.sorted((o1, o2) -> {//有状态操作，无法并行
					System.out.println(Thread.currentThread().getName() + " sort: " + o1 + ":" + o2);
					return o1 - o2;
				})
				.map(i -> {
					System.out.println(Thread.currentThread().getName() + " peek:" + i);
					return "" + i;
				})
				.parallel()
            	.count();//最终操作
	}
```

输出：

```shell
ForkJoinPool.commonPool-worker-31 peek:-363772240	#
ForkJoinPool.commonPool-worker-5 peek:1545317691
ForkJoinPool.commonPool-worker-19 peek:-1760380366
ForkJoinPool.commonPool-worker-9 peek:1208561582
ForkJoinPool.commonPool-worker-31 filter:-363772240	#
ForkJoinPool.commonPool-worker-5 filter:1545317691
ForkJoinPool.commonPool-worker-19 filter:-1760380366
main peek:-1986972922
main filter:-1986972922
ForkJoinPool.commonPool-worker-9 filter:1208561582
ForkJoinPool.commonPool-worker-23 peek:-1057894078
ForkJoinPool.commonPool-worker-23 filter:-1057894078
ForkJoinPool.commonPool-worker-13 peek:271304166
ForkJoinPool.commonPool-worker-27 peek:-1549884799
ForkJoinPool.commonPool-worker-27 filter:-1549884799
ForkJoinPool.commonPool-worker-13 filter:271304166
ForkJoinPool.commonPool-worker-21 peek:-1257783052
ForkJoinPool.commonPool-worker-17 peek:-1514737808
ForkJoinPool.commonPool-worker-21 filter:-1257783052
ForkJoinPool.commonPool-worker-17 filter:-1514737808
main sort: 271304166:1208561582	#
main sort: 1545317691:271304166
main sort: 1545317691:1208561582
```



