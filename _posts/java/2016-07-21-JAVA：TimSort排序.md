---
categories:
  - Technology
tags:
  - Java
---

# 零 楔子
我们大部分人第一次听说TimSort一般是和“java.lang.IllegalArgumentException: Comparison method violates its general contract!”这个异常有关。在Java7中排序默认使用了TimSort排序，这要求提供的比较函数Comparator函数必须明确返回等于的情况，而在java7之前是没这样要求的，因此从java7- 迁移到java7时或者按照老方式不为Comparator提供明确等情况时就可能触发上面的异常。关于这个异常更深入的原因可以参考：[图解JDK7的Comparison method violates its general contract异常](http://www.tuicool.com/articles/MZreyuv)，本文不再对此原因进行分析

TimSort最初应该是在Python中引入的，然后java7中也引用了，JDK宁愿为了抛弃部分的向下兼容性也要引入TimSort，说明该排序算法一定有其“过人之举”

<img src="http://o9l56z0kf.bkt.clouddn.com/image/blog/TimSort-%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95%E5%AF%B9%E6%AF%94.png" />

||最好|最差|
|---|----|----|
|时间|O(n)|O(nlog(n))|
|内存|常量|n/2|

TimSort本质上可以说是MergeSort和BinarySort的结合，它通过对MergeSort的 ___一点点累计起来的细节优化___，成就了它的“过人之举”。

>“Why not a short step to a thousand miles”   
&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;一句古老的中国名言

下面就通过TimSort一点点优化的过程来感受下这句至理名言，我们主要分为【子序列的选择】和【序列的归并】这两大部分来说明。

# 一 子序列的选取

## 跬步1：有序子序列

MergeSort本身并没有考虑原有序列的排列情况，TimSort设法把序列中已有的有序子序列给利用了起来，TimSort中把有序的一段子序列定义为run

>从【某个位置开始】的一段【__最长不递减子序列__】或者一段【__最长严格递减子序列__】

对于递减子序列会通过从两头交换直至中间的方式转换为递增子序列，因为两头交换对于相等的数据来说会破坏【__稳定性__】，所以这里要求必须是【__严格递减__】的子序列。

注意：对于发现的run，会记录其初始位置和长度到一个栈中，然后从栈顶取run来判断是不是能够进行归并，不是所有的run都查找之后才开始进行merge的，而是一边查找一边merge的，后面会说。

查找run的代码可以参考如下：

```java
private static <T> int countRunAndMakeAscending(T[] a, int lo, int hi,
                                                Comparator<? super T> c) {
    assert lo < hi;
    int runHi = lo + 1;
    if (runHi == hi)
        return 1;

    // Find end of run, and reverse range if descending
    if (c.compare(a[runHi++], a[lo]) < 0) { // Descending
        while (runHi < hi && c.compare(a[runHi], a[runHi - 1]) < 0)
            runHi++;
        reverseRange(a, lo, runHi);
    } else {                              // Ascending
        while (runHi < hi && c.compare(a[runHi], a[runHi - 1]) >= 0)
            runHi++;
    }

    return runHi - lo;
}
```

## 跬步2：run的最小长度

对于比较“任性Random”的数据可能很难找到一段比较长的run，为了加以改善，防止退化为基本的mergeSort，TimSort允许分段进行排序后形成有序的序列，并规定了序列允许的最短的长度minRun

>为了方便后续的排序，每段run的长度不能少于minRun，如果数据本身无法达到minRun长度有序，就通过二分插入排序的方法使序列达到minRun的长度

minRun初始值一般选择64(c版本中)或者32(java版本中)，太大如256会增大二分插入的开销，太小如8会增加函数调用等的开销，__但是__，minRun并不是固定的。

## 跬步3：minRun的选取

假如序列总长度为2112，minRun选择32，序列数据高度随机化，那么如果假设每个run长度都为32，divmod(2112, 32) = (66, 0), 总计66个run, 根据后面的merge过程可以知道，最开始的64个run会合并，并且是【__完美归并__】,后面的两个RUN会一块合并，最后剩余两个2048长和64长的两个run进行归并，这并不是最好的结果。

>如果待归并的两个子序列长度一样，被称为完美归并(perfect merge)

考虑如果取minRun为33，divmod(2112, 33) = (64, 0)，那么这64个run都会进行完美归并,因此这种情况下33是比32更优的选择。

因此最佳的选择应该是在(32,65)(java版本中是[16, 32])中取一个minRun,(q, r) = divmod(N, minrun),使得q是2的幂次，或者小于但很接近2的幂次，如果拿出纸笔(或者利用脑容量)证明一下，满足这样的结果就是：取N的高6位(java实现中是高4位)，如果其他位不全为0，就再加1

下面是java中获取minRun的代码

```java
/**
 * Returns the minimum acceptable run length for an array of the specified
 * length. Natural runs shorter than this will be extended with
 * {@link #binarySort}.
 *
 * Roughly speaking, the computation is:
 *
 *  If n < MIN_MERGE, return n (it's too small to bother with fancy stuff).
 *  Else if n is an exact power of 2, return MIN_MERGE/2.
 *  Else return an int k, MIN_MERGE/2 <= k <= MIN_MERGE, such that n/k
 *   is close to, but strictly less than, an exact power of 2.
 *
 * For the rationale, see listsort.txt.
 *
 * @param n the length of the array to be sorted
 * @return the length of the minimum run to be merged
 */
private static int minRunLength(int n) {
    assert n >= 0;
    int r = 0;      // Becomes 1 if any 1 bits are shifted off
    while (n >= MIN_MERGE) {
        r |= (n & 1);
        n >>= 1;
    }
    return n + r;
}
```

# 二 序列的归并

## 跬步4：Run合并的时机

假设有A,B,C连续三段的run，可以选择A和B合并，也可以选择B和C合并，但不能选择A和C合并，否则会破坏稳定性，为了取得比较好的合并效果，如果下面的两个规则都不满足则进行合并，如果都满足可以先不合并，继续寻找下一个序列run

>1. A > B + C
2. B > C

第二条规则说明了栈中从栈底到栈顶run的长度是递减的，第一条限制了栈中的未merge的run的长度最快也就以Fibonacci数列进行增长，这样就保证了栈的长度不会超过log(1.618)(N)

归并的时候从栈顶取数据，如果只有两个run，判断是否B>C，满足不进行归并，不满足则进行合并，如果有超过三个run，满足A <= B + C，就选择A和C中的小者和B进行合并，不满足则不进行归并，归并后的新序列run也要放入栈中。要保证【__栈中所有相邻的三个序列长度都满足上面两条规则__】，后面会看到java中TimSort曾经的bug会和这有关。

考虑下假设长度为：128, 64, 32, 16, 8, 4, 2, 2时如何进行合并的？

一切的目的都是为了使得进行merge的两个run的长度能够尽量一致，同时保持栈中记录的run的数量能够尽可能的少

## 插曲：一览众山小

在看具体的归并过程之前，可以盗图来先看下整个的大致流程了，图片来自[http://www.freebuf.com/vuls/62129.html](http://www.freebuf.com/vuls/62129.html)

a. 得到最早的两个run，但不满足runLen[0]>runLen[1],进行merge后成为新的run[0]

![TimSort](http://o9l56z0kf.bkt.clouddn.com/image/blog/TimSort-%E6%BC%94%E7%A4%BA%E6%AD%A5%E9%AA%A41.png)

b. 得到run[1]满足runLen[0] > runLen[1]不进行merge，继续得到run[2]满足runLen[0] > runLen[1] + runLen[2]和runLen[1]>runLen[2]也不进行merge

![TimSort](http://o9l56z0kf.bkt.clouddn.com/image/blog/TimSort-%E6%BC%94%E7%A4%BA%E6%AD%A5%E9%AA%A42.png)

c. 最后得到run[3],不满足runLen[1] > runLen[2] + runLen[3],进行merge，后续一直merge到只剩1个run就是最终结果了

![TimSort](http://o9l56z0kf.bkt.clouddn.com/image/blog/TimSort-%E6%BC%94%E7%A4%BA%E6%AD%A5%E9%AA%A43.png)

## 跬步5：Merge内存的开销

对于Merge操作，理论上可以得到常量开销，但比较麻烦而且速度比较慢，实际操作中都会使用额外的内存空间。额外内存空间的选择只要是要合并的序列的最短的那个就行，但是还可以更少。

对于如下图所示的A和B两段连续序列，会先二分查找B[0]在A中的位置，以及A[-1]在B中的位置，查找之后可以得知A中需要移动的只是4~6共4个数据，B中需要移动的是3~5共3个数据，因此临时内存段选择3个数据的长度就可以了。
![MergeMemory](http://o9l56z0kf.bkt.clouddn.com/image/blog/TimSort-MergeMemory.png)

单单来看这个二分查找的过程可能带来额外的开销，但整的来说无论在时间还是内存上一般都能够带来更大的好处的。

根据选取的temp内存段的不同，开始进行merge_low或者merge_high操作，如上图中temp段是B左边一段的复制，这时进行merge_high操作，反之，如果temp是取得A中右侧的一段，会进行merge_low操作。

## 跬步6：Merge的过程

无论merge_low还是merge_high，都进行我们平常所熟知的merge操作，比如上面的图中应该是merge_high,temp[2] < A[5]应该把A[5]放入B[2]的地方，实际操作中因为上面搜索的原因已经明确了temp[2]<A[5]所以是可以省略掉这次比较直接移动，从下一组比较开始。

one pair at a time模式

>依次比较A和B中的元素，根据比较的结果取一个数据放到应该放置的地方，这种方式称为“one pair at a time”

简单的说就是如果同一个run连续多个能够移动，那就可以以“块”的形式一起移动，而不需要单个的慢慢移动了，因此在进行比较移动的同时，会记录从同一个run连续移动的次数，如果达到阈值MIN_GALLOP，就会进入galloping mode。如果两个run能够连续移动的元素个数少于MIN_GALLOP个的话就会重新回到one pair at a time模式下。


galloping 模式

>不同于one pair at a time模式下依次进行比较每个元素，galloping模式会先以指数地址再以二分查找的方式快速的定位到元素应该在的位置，然后以块的形式批量移动数据，这种“跳跃舞动”的方式称为galloping 模式

以A,B进行举例，首先查找A[0]在B中应该的位置，会用A[0]依次和B[0], B[1],B[3],B[7],...,B[2^j - 1],...进行比较，直到找到k满足B[2^(k-1) - 1] < A[0] <= B[2^k - 1]，然后再在这段范围内进行二分查找。

galloping和二分查找相结合的方式需要的比较次数最多为2*lg(B)；而直接使用二分查找的方式无论A[0]位置在哪都是lg(B+1)次比较。除非A[0]在B很靠后的地方，而且B的长度很长，一般情况下galloping加二分查找的方式能够更快。

但是并不是所有情况下galloping总比one pair at a time好的

```
index in B where    # compares linear  # gallop  # binary  gallop
A[0] belongs        search needs       compares  compares  total
----------------    -----------------  --------  --------  ------
               0                    1         1         0       1

               1                    2         2         0       2

               2                    3         3         1       4
               3                    4         3         1       4

               4                    5         4         2       6
               5                    6         4         2       6
               6                    7         4         2       6
               7                    8         4         2       6

               8                    9         5         3       8
               9                   10         5         3       8
              10                   11         5         3       8
              11                   12         5         3       8
```

从上面表格可以看出从i=6开始之后，gallop模式才一直取得上风，所以MIN_GALLOP一般取值为7。

## 跬步7：变化的MIN_GALLOP

但是数据不总是尽如人意，阈值也不能总是为7，算法设计了一个动态调整的minGallop，初始值为MIN_GALLOP，算法在galloping模式下待的越久，minGallop越小，如果算法在one pair at a time模式下运行的更久，则minGallop会变得越来越大。这种自动调价的方式更能够从容的适应各种“恶劣的数据环境”。


至此TimSort算法已经介绍完毕了，已经达千里之遥了~

# 三 曾经的bug

成长的道路难免磕磕碰碰，java中的TimSort也是曾经bug缠身的。

前面没有细说栈的选择，只提到栈的长度不会超过log(1.618)(N)，c的实现版本中提供了一个足够大的值，而java版本中稍微进行了一下优化，目前的java版本中栈的长度定义如下：

```java
/*
 * Allocate runs-to-be-merged stack (which cannot be expanded).  The
 * stack length requirements are described in listsort.txt.  The C
 * version always uses the same stack length (85), but this was
 * measured to be too expensive when sorting "mid-sized" arrays (e.g.,
 * 100 elements) in Java.  Therefore, we use smaller (but sufficiently
 * large) stack lengths for smaller arrays.  The "magic numbers" in the
 * computation below must be changed if MIN_MERGE is decreased.  See
 * the MIN_MERGE declaration above for more information.
 */
int stackLen = (len <    120  ?  5 :
                len <   1542  ? 10 :
                len < 119151  ? 24 : 40);
runBase = new int[stackLen];
runLen = new int[stackLen];
```

但是它最初却不是这样的，而是

```java
int stackLen = (len <    120  ?  5 :
                len <   1542  ? 10 :
                len < 119151  ? 19 : 40);
runBase = new int[stackLen];
runLen = new int[stackLen];
```

上面的栈长度值的选择是有理论依据的，这里不再详述，如果想了解可以参考[http://www.freebuf.com/vuls/62129.html](http://www.freebuf.com/vuls/62129.html)

另外对栈顶的run长度的判断代码如下：

```java
/**
 * Examines the stack of runs waiting to be merged and merges adjacent runs
 * until the stack invariants are reestablished:
 *
 *     1. runLen[i - 3] > runLen[i - 2] + runLen[i - 1]
 *     2. runLen[i - 2] > runLen[i - 1]
 *
 * This method is called each time a new run is pushed onto the stack,
 * so the invariants are guaranteed to hold for i < stackSize upon
 * entry to the method.
 */
private void mergeCollapse() {
    while (stackSize > 1) {
        int n = stackSize - 2;
        if (n > 0 && runLen[n-1] <= runLen[n] + runLen[n+1]) {
            if (runLen[n - 1] < runLen[n + 1])
                n--;
            mergeAt(n);
        } else if (runLen[n] <= runLen[n + 1]) {
            mergeAt(n);
        } else {
            break; // Invariant is established
        }
    }
}
```

上面的代码判断实际上只判断了栈顶的三个元素，如果栈中run长度为120, 80, 25, 20, 30，合并之后变为120, 80, 45, 30，这时候栈顶的三个元素80，45，30是满足条件的，但是120，80，45是不满足的。这就会导致栈的长度超出理论值,从而出现java.lang.ArrayIndexOutOfBoundsException异常。

# 四 二分插入排序

小数据量的情况下通常使用二分插入排序，因此二分插入排序是很多其他排序的基础，下面摘录了下java版TimSort排序中用到的二分插入排序代码，但不再进行详说，可以自行思考和关注几点
1. 开始位置是直接从start开始的，利用了查找run过程中得到的start数据进行优化
2. 二分查找while循环过程中，为什么不使用c.compare(pivot, a[mid]) == 0时直接跳出循环？==> 为了排序的稳定性
3. 移动元素时利用System.arraycopy进行优化

```java
@SuppressWarnings("fallthrough")
private static <T> void binarySort(T[] a, int lo, int hi, int start,
                                   Comparator<? super T> c) {
    assert lo <= start && start <= hi;
    if (start == lo)
        start++;
    for ( ; start < hi; start++) {
        T pivot = a[start];

        // Set left (and right) to the index where a[start] (pivot) belongs
        int left = lo;
        int right = start;
        assert left <= right;
        /*
         * Invariants:
         *   pivot >= all in [lo, left).
         *   pivot <  all in [right, start).
         */
        while (left < right) {
            int mid = (left + right) >>> 1;
            if (c.compare(pivot, a[mid]) < 0)
                right = mid;
            else
                left = mid + 1;
        }
        assert left == right;

        /*
         * The invariants still hold: pivot >= all in [lo, left) and
         * pivot < all in [left, start), so pivot belongs at left.  Note
         * that if there are elements equal to pivot, left points to the
         * first slot after them -- that's why this sort is stable.
         * Slide elements over to make room for pivot.
         */
        int n = start - left;  // The number of elements to move
        // Switch is just an optimization for arraycopy in default case
        switch (n) {
            case 2:  a[left + 2] = a[left + 1];
            case 1:  a[left + 1] = a[left];
                     break;
            default: System.arraycopy(a, left, a, left + 1, n);
        }
        a[left] = pivot;
    }
}
```

# 五 站在巨人的肩膀上

没有下面的参考，就没有本文：

* [listsort](http://svn.python.org/projects/python/trunk/Objects/listsort.txt)
* [OpenJDK’s java.utils.Collection.sort() is broken:The good, the bad and the worst case](http://envisage-project.eu/wp-content/uploads/2015/02/sorting.pdf)
* [如何找出Timsort算法和玉兔月球车中的Bug？](http://www.freebuf.com/vuls/62129.html)
* [图解JDK7的Comparison method violates its general contract异常](http://www.tuicool.com/articles/MZreyuv)
