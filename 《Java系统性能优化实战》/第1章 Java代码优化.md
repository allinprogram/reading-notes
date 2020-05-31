# 第1章 Java代码优化

## 可优化的代码

反面教材

``` java
public Map buildArea(List<Area> areas) {
    if(areas.isEmpty()) {
        // 此处不要使用Collections.EMPTY_MAP进行极致优化；
        // 其无法进行写操作，会抛出UnsupportedOperationException；
        // 并且某些微服务框架无法正确识别和序列化EMPTY_MAP；
        // 在JDK8中，创建一个空的HashMap()代价非常小。
        return new HashMap();
    }
    Map<String, Area> map = new HashMap<>();
    for(Area area : areas) {
        String key = area.getProvinceId() + "#" + area.getCityId();
        map.put(key, area);
    }
    return map;
}
```

可优化点

1. 返回值为Map，代码不易阅读；

    解：应该加上泛型`Map<String, Area>`。

2. 上述代码中的参数`key`以String描述，阅读不易，且不利于项目改动；

    解：用一个CityKey对象替代`String`类型的`key`，并在`Area`中维护`public CityKey buildKey()`方法，这样任何`CityKey`含义变更、重构都不会影响到使用它的代码。

修改后的代码

``` java
@Getter
@Setter
@EqualsAndHashCode
@AllArgsConstructor
public class Citykey {
    private Integer provinceId;
    private Integer cityId;
}
```

``` java
public class Area {
    // ...
    public CityKey buildKey() {
        return new CityKey(provinceId, cityId);
    }
}

```

``` java
public Map<CityKey, Area> buildArea(List<Area> areas) {
    if(areas.isEmpty()) {
        return new HashMap();
    }
    Map<CityKey, Area> map = new HashMap<>();
    for(Area area : areas) {
        map.put(area.buildKey(), area);
    }
    return map;
}
```

书中给出了一个思考：构造一个新的`CityKey`对象会不会导致性能降低？

解：不会，在上述参数`key`的字符串构造中，整型转字符串会非常耗时，这才是耗时大头。通过Debug可以发现，字符串拼接的时候，会使用到`StringBuilder`，其最终会调用`Integer.toString()`方法：

``` java
public static String toString(int i) {
    if (i == Integer.MIN_VALUE)
        return "-2147483648";
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
    char[] buf = new char[size];
    getChars(i, size, buf);
    return new String(buf, true);
}
```

首先通过`Integer.stringSize()`确定字符串长度：

``` java
final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
                                      99999999, 999999999, Integer.MAX_VALUE};
// Requires positive x
static int stringSize(int x) {
    for (int i=0; ; i++)
        if (x <= sizeTable[i])
            return i+1;
}
```

获取到长度后，通过`Integer.getChars()`完成整型到字符串的转化：

``` java
static void getChars(int i, int index, char[] buf) {
    int q, r;
    int charPos = index;
    char sign = 0;

    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // Generate two digits per iteration
    while (i >= 65536) {
        q = i / 100;
    // really: r = i - (q * 100);
        r = i - ((q << 6) + (q << 5) + (q << 2));
        i = q;
        buf [--charPos] = DigitOnes[r];
        buf [--charPos] = DigitTens[r];
    }

    // Fall thru to fast mode for smaller numbers
    // assert(i <= 65536, i);
    for (;;) {
        q = (i * 52429) >>> (16+3);
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
        buf [--charPos] = digits [r];
        i = q;
        if (i == 0) break;
    }
    if (sign != 0) {
           buf [--charPos] = sign;
    }
}
```

完成整型到字符串的转换后，并没有真正结束。接来下的过程中会执行`AbstractStringBuilder.append()`：

``` java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}
```

其中`ensureCapacityInternal()`方法用于保证`StringBuilder`的`buf`足够长，如果长度不够，则进行扩容。这意味着需要新建一份较大的内存块，然后将原有的内容复制到这份较大的内存块中，这也是一个消耗资源的地方。

说到这儿，不得不提一个native方法`System.arraycopy()`，其主要负责将原有内容复制到新内存块中。

`str.getChars()`也会做一次内存复制，将`str`的内容复制到新扩容的`buf`中，至此，一个字符串的拼接操作才算真正完成。

## 性能监控

jvisualvm的使用

## JMH

R大提到过，进行Java程序性能分析最起码也得使用JMH，而不是`System.currentTimeMillis()`。
