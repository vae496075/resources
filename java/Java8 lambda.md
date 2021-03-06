###### 1. Collector
```
/**
 * 自定义mapper的Collector
 *
 * @param resultMapper resultMapper
 * @param <T>          <T>
 * @param <R>          <R>
 * @return list
 */
public static <T, R> Collector<T, ?, List<R>> mapperCollector(Function<T, R> resultMapper) {
    BiConsumer<List<R>, T> biConsumer = (list, i) -> list.add(resultMapper.apply(i));
    BinaryOperator<List<R>> combiner = (a, b) -> {
        a.addAll(b);
        return a;
    };
    return Collector.of(ArrayList::new, biConsumer, combiner, i -> i);
}


/**
* collector取一个（最后一个）
*
* @param secondMapper
* @param resultMapper
* @param <T>
* @param <S>
* @param <R>
* @return Collector
*/
private static <T, S, R> Collector<T, ?, Map<S, R>> collectorSingle(Function<T, S> secondMapper
		, Function<T, R> resultMapper) {
	return Collectors.toMap(secondMapper, resultMapper, (key1, key2) -> key2);
}


/**
* collector取集合
*
* @param compareMapper
* @param <T>
* @param <S>
* @return
*/
public static <T, S> Collector<T, ?, Map<S, List<T>>> collectorList(Function<T, S> compareMapper) {
    return Collectors.groupingBy(compareMapper, Collectors.toList());
}


/**
* customize collector
*/
Collectors.of(Supplier<A> supplier,
                BiConsumer<A, T> accumulator,
                BinaryOperator<A> combiner,
                Function<A, R> finisher,
                Characteristics... characteristics)


/**
*  group by multiple fields(single level map)
*/
public static <T> Map<String, List<T>> groupByMultipleField(List<T> data, List<Function<T, ?>> groupFields) {
	Function<T, String> groupFunction = i -> {
		StringBuilder key = new StringBuilder();
		groupFields.forEach(v -> key.append(v.apply(i)));
		return key.toString();
	};
	if (data != null) {
		return data.stream().collect(Collectors.groupingBy(groupFunction));
	} else {
		return new HashMap<>(0);
	}
}


/**
*  group by multiple fields(multiple level map)
*/
public static <F, S, T> Map<F, Map<S, List<T>>> groupByMultipleField(List<T> data, Function<T, F> groupField1, Function<T, S> groupField2) {
	return data.stream().collect(Collectors.groupingBy(groupField1, Collectors.groupingBy(groupField2)));
}

/**
 * 去重
 *
 * @param list  origin list
 * @param equal equal function
 * @param value value function
 * @param <T>   T
 * @param <R>   R
 * @param <V>   V
 * @return 去重之后的集合
 */
public static <T, R, V> List<V> distinct(List<T> list, Function<T, R> equal, Function<T, V> value) {
    if (list == null) {
        return new ArrayList<>();
    } else {
        Map<R, V> collect = getNonEmptyList(list).stream().filter(i -> equal.apply(i) != null).collect(Collectors.toMap(equal, value, (a, b) -> a));
        return Optional.of(collect).map(Map::values).map(ArrayList::new).orElse(new ArrayList<>());
    }
}
```

###### 2. Comparable

```
/**
* 排序（允许空值）
*
* @param list          list
* @param compareMapper compareMapper
* @param resultMapper  resultMapper
* @param sort          DESC或者ASC（lambda默认为ASC）
* @param <T>           <T>
* @param <R>           <R>
* @return list
*/
public static <T, R, U extends Comparable<? super U>> List<R> sort(List<T> list, Function<T, U> compareMapper
        , Function<T, R> resultMapper, Sort.Direction sort) {
    Comparator<? super T> comparator = Comparator
        .comparing(compareMapper, Comparator.nullsFirst(Comparator.naturalOrder()));
    comparator = sort == Sort.Direction.DESC ? comparator.reversed() : comparator;
    return getNonEmptyList(list).stream().sorted(comparator).map(resultMapper).collect(Collectors.toList());
}
```

###### 3. Predicate

```
/**
* batch filter
*
* @param sourceList sourceList
* @param listFilter listFilter
* @param <T>        <T>
* @return list
*/
public static <T> List<T> filter(List<T> sourceList, List<Predicate<T>> listFilter) {
    return Optional.of(sourceList).map(list -> {
        Stream<T> stream = list.stream();
        for (Predicate<T> filter : listFilter) {
            stream = stream.filter(filter);
        }
        return stream.collect(Collectors.toList());
    }).orElse(new ArrayList<>());
}


```

###### 3. Function
```
/**
* return i->i
*/
Function.identity();
```
###### 4. Others
```
//	stream to array(primitive data type)
Integer[] array = Arrays.asList(1, 2, 3).stream().mapToInt(i->i).boxed().toArray(Integer[]::new);
//	stream to array(wapper class)
int[] arrayValue = Arrays.asList(1, 2, 3).stream().mapToInt(i -> i).toArray();
```
###### 5. Page for list

```java
/**
 * page
 *
 * @param list list
 * @param size page size
 * @param <T>  T
 * @return List<List < T>>
 */
public static <T> List<List<T>> page(List<T> list, int size) {
    list = list == null ? new ArrayList<>() : list;
    List<T> finalList = list;
    return IntStream.range(0, list.size()).mapToObj(i -> new IndexObject<T>(i, finalList.get(i))).collect(collectorByPage(size));
}

/**
 * page
 *
 * @param size page size
 * @param <T>  T
 * @return List<List < T>>
 */
private static <T> Collector<IndexObject<T>, Map<Integer, List<T>>, List<List<T>>> collectorByPage(int size) {
    Supplier<Map<Integer, List<T>>> supplier = HashMap::new;
    BiConsumer<Map<Integer, List<T>>, IndexObject<T>> accumulator = (map, o) -> {
        int key = o.getIndex() / size;
        List<T> temp = map.get(key);
        temp = temp == null ? new ArrayList<>() : temp;
        temp.add(o.getValue());
        map.put(key, temp);
    };
    BinaryOperator<Map<Integer, List<T>>> combiner = (a, b) -> {
        a.forEach((index, value) -> {
            List<T> temp = b.get(index);
            temp = temp == null ? new ArrayList<>() : temp;
            temp.addAll(value);
            b.put(index, temp);
        });
        return b;
    };
    Function<Map<Integer, List<T>>, List<List<T>>> finisher = map -> new ArrayList<>(map.values());
    return Collector.of(supplier, accumulator, combiner, finisher);
}

/**
 * object with index
 *
 * @param <T>
 */
@Data
@Accessors(chain = true)
@NoArgsConstructor
@AllArgsConstructor
private static final class IndexObject<T> {
    private Integer index;
    private T value;
}
```

