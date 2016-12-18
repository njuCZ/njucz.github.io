title: Java默认排序算法问题
date: 2015-03-23 14:26:24
tags: [java, jdk, sort]
---

# 前言
虽然学习Java已经有3、4年时间了，不过一直没有想过要看jdk的源码，导致面试官问Java默认排序算法的实现时只能吞吞吐吐，颇为难堪。遂将相关的知识整理下，警醒自我。
注：本文是依据 jdk 1.7 的源码进行说明

# java 默认排序算法的实现
打开java.util包中的Arrays类文件，我们可以看到其sort方法主要是分以下三种情况分别进行实现 

## java 基本类型的排序
在Arrays类中使用方法重载提供了对int、long、short、char、byte、float、double型数组的排序，其实现都是通过调用DualPivotQuicksort.sort()方法。在DualPivotQuicksort类中，我们可以看到如下代码：

	/**
     * If the length of an array to be sorted is less than this
     * constant, Quicksort is used in preference to merge sort.
     */
    private static final int QUICKSORT_THRESHOLD = 286;

    /**
     * If the length of an array to be sorted is less than this
     * constant, insertion sort is used in preference to Quicksort.
     */
    private static final int INSERTION_SORT_THRESHOLD = 47;

其含义即为若排序的数组个数超过QUICKSORT_THRESHOLD，那么就使用merge sort；个数位于QUICKSORT_THRESHOLD和INSERTION_SORT_THRESHOLD之间就使用Quicksort；个数少于INSERTION_SORT_THRESHOLD那么就使用插入排序。

下面我们以int类型数组的排序简单介绍下其源码，并验证上述的论述（源文件107行 ~ 191行）。

首先程序判断排序的个数是否超过QUICKSORT_THRESHOLD

	if (right - left < QUICKSORT_THRESHOLD) {
        sort(a, left, right, true);     // Use Quicksort on small arrays
        return;
    }

然后程序试图用MAX_RUN_COUNT数量个run数组，去将待排序数组划分成一个个递增的序列（即从前到后寻找递增或递减的序列，并将递减的序列排序成递增的，然后在run数组中保存临界下标值）。如果发现序列的个数超过MAX_RUN_COUNT,就表示待排序的数组是高度不规则化的，那么程序还是会使用快排去进行排序。

		// Check if the array is nearly sorted
        for (int k = left; k < right; run[count] = k) {
            if (a[k] < a[k + 1]) { // ascending
                while (++k <= right && a[k - 1] <= a[k]);
            } else if (a[k] > a[k + 1]) { // descending
                while (++k <= right && a[k - 1] >= a[k]);
                for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                    int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
                }
            } else { // equal
                for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                    if (--m == 0) {
                        sort(a, left, right, true);
                        return;
                    }
                }
            }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
            if (++count == MAX_RUN_COUNT) {
                sort(a, left, right, true);
                return;
            }
        }
若经过上述的步骤发现待排序的数组不是高度不规则化的，那么就会进行merge排序（之前已经将带排序数组分成了一个个小的递增序列，所以此时只需进行merge操作就可以）。

上述判断了程序是使用Quicksort还是merge sort，而在quicksort的实现中又针对小数目的数组进行插入排序进行了优化。

## java 对象类型的排序(老版本)
在jdk 1.6及以前的版本中，对于object类型的数组默认是使用merge sort，而在jdk 1.7中将其替换为下文将要介绍的Timsort，但是于此同时，为了保留jdk的兼容性，merge sort的实现并没有被立即删掉，不过已经注明在未来的版本中将被移除。

如今，为了使用jdk 1.6的排序算法，需要在启动参数中（例如eclipse.ini）添加-Djava.util.Arrays.useLegacyMergeSort=true。这可以通过Arrays.java中的下述语句来理解。
    
    /**
     * Old merge sort implementation can be selected (for
     * compatibility with broken comparators) using a system property.
     * Cannot be a static boolean in the enclosing class due to
     * circular dependencies. To be removed in a future release.
     */
    static final class LegacyMergeSort {
        private static final boolean userRequested =
            java.security.AccessController.doPrivileged(
                new sun.security.action.GetBooleanAction(
                    "java.util.Arrays.useLegacyMergeSort")).booleanValue();
    }

    public static void sort(Object[] a) {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a);
        else
            ComparableTimSort.sort(a);
    }

通过上述代码可知，旧版本的对象数组排序是通过legacyMergeSort()方法进行实现的。而这个方法内部则是通过merge sort处理大量对象和插入排序处理少量对象完成的。具体可以阅读源码或参考文章[java类库中Arrays的排序算法探析（Object与泛型类型）](http://jiangshuiy.iteye.com/blog/1449935)

## java 对象类型的排序(新版本)
通过上一节的说明，我们知道目前java对于对象的默认排序是使用名为timsort的算法。timsort算法的基本思想和老版本的还是一致的，即将归并排序和插入排序结合，并进行了优化。但其基本思想是将待排序数组划分成一个个递增的run序列（将递减的排序成递增的，即递增子序列或递减子序列为一个run），并对run序列进行合并，该合并方法名为binarySort，即采用的是binary search的方式查找插入位置。由于待排序数组可能会产生很多的run序列，程序中会采用一个栈来保存所有的run序列，并在条件满足时进行run序列的合并。

针对timsort具体的实现及说明可以参考文章[OpenJDK 源代码阅读之 TimSort](http://blog.csdn.net/on_1y/article/details/30109975)以及[Timsort Wiki](http://en.wikipedia.org/wiki/Timsort)

###  JDK 1.7 中更换排序算法后可能引发的问题
使用新版本的jdk排序算法可能会遇到如下的异常：

    Exception in thread "main" java.lang.IllegalArgumentException: Comparison method violates its general contract!

这是因为Timsort对对象间比较的实现要求更加严格，其Comparator的实现必须保证以下几点：
1. sgn(compare(x, y)) == -sgn(compare(y, x)) 
2. (compare(x, y)>0) && (compare(y, z)>0) 意味着 compare(x, z)>0 
3. compare(x, y)==0 意味着对于任意的z：sgn(compare(x, z))==sgn(compare(y, z)) 均成立

而我们的代码中，某个compare()实现片段是这样的：

    public int compare(ComparatorTest o1, ComparatorTest o2) { 
        return o1.getValue() > o2.getValue() ? 1 : -1; 
    }

这就违背了a)原则：假设X的value为1，Y的value也为1；那么compare(X, Y) ≠ –compare(Y, X) 
PS: TimSort不仅内置在各种JDK 7的版本，也存在于Android SDK中（尽管其并没有使用JDK 7）。

因为常见的解决方案就是如下三种：
1. 更改内部实现：例如对于上个例子，就需要更改为
        public int compare(ComparatorTest o1, ComparatorTest o2) { 
            return o1.getValue() == o2.getValue() ? 0 :  
                    (o1.getValue() > o2.getValue() ? 1 : -1); 
        }

2.  Java 7预留了一个接口以便于用户继续使用Java 6的排序算法：在启动参数中（例如eclipse.ini）添加-Djava.util.Arrays.useLegacyMergeSort=true

3. 将这个IllegalArgumentException手动捕获住（不推荐）