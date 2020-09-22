---
categories: algorithm
layout: post
---

- Table
{:toc}

# Java的基础运算性能

在做时间复杂度分析之前，一般都会默认所有的基础运算的时间复杂度都是$O(1)$。这里的基础运算包括加减乘除，函数调用，位运算等，逻辑运算，分支跳转，内存访问等。

下面我们来具体计算一下每个运算的常数大小。

下面是我的CPU信息：

```
Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
```

下面是我的JVM信息。

```
openjdk version "1.8.0_265"
OpenJDK Runtime Environment (build 1.8.0_265-8u265-b01-0ubuntu2~20.04-b01)
OpenJDK 64-Bit Server VM (build 25.265-b01, mixed mode)
```

下面是我的JVM参数

```
-XX:TieredStopAtLevel=1
```

下面是测试代码：

```java
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) {

        int time = 10;
        Test obj = new Test((int) 1e8);
        for (int i = 0; i < time; i++) {
            System.out.println("round " + i + "...");
            test("empty", obj::empty);
            test("plus", obj::plus);
            test("plusMultiway", obj::plusMultiway);
            test("longPlus", obj::longPlus);
            test("subtract", obj::subtract);
            test("mul", obj::mul);
            test("longMul", obj::longMul);
            test("div", obj::div);
            test("mod", obj::mod);
            test("invoke", obj::invoke);
            test("and", obj::and);
            test("choose", obj::choose);
            test("doublePlus", obj::doublePlus);
            test("doubleMul", obj::doubleMul);
            test("doubleDiv", obj::doubleDiv);
        }

        for (String key : elapse.keySet()) {
            System.out.println(key + " finished in " + Math.round(elapse.get(key) / (double)time) + "ms");
        }
    }

    private static Map<String, Long> elapse = new HashMap<>();

    public static void test(String name, Runnable task) {
        long now = System.currentTimeMillis();
        task.run();
        long end = System.currentTimeMillis();
        elapse.put(name, elapse.getOrDefault(name, 0L) + end - now);
    }

    int round;

    public Test(int round) {
        this.round = round;
    }

    public Test() {
        this((int) 1e9);
    }


    public void plus() {
        int sum = 0;
        for (int i = 1; i <= round; i++) {
            sum += i;
        }
    }

    public void plusMul() {
        int sum = 0;
        for (int i = 1; i <= round; i++) {
            sum += i;
            sum *= i;
        }
    }

    public void choose() {
        int sum = 0;
        for (int i = 1; i <= round; i++) {
            if (sum < 0) {
                sum += i;
            } else {
                sum -= i;
            }
        }
    }

    public void plusMultiway() {
        int sum = 0;
        for (int i = 1; i <= round; i += 2) {
            sum += i;
            sum += i + 1;
        }
    }


    public void subtract() {
        int sum = 0;
        for (int i = 1; i <= round; i++) {
            sum -= i;
        }
    }


    public void mul() {
        int sum = 1;
        for (int i = 1; i <= round; i++) {
            sum *= i;
        }
    }


    public void longMul() {
        long sum = 1;
        for (int i = 1; i <= round; i++) {
            sum *= i;
        }
    }

    public void longPlus() {
        long sum = 1;
        for (int i = 1; i <= round; i++) {
            sum += i;
        }
    }


    public void div() {
        int sum = 1;
        for (int i = 1; i <= round; i++) {
            sum /= i;
        }
    }


    public void mod() {
        int sum = 1;
        for (int i = 1; i <= round; i++) {
            sum %= i;
        }
    }


    public void empty() {
        int sum = 1;
        for (int i = 1; i <= round; i++) {
        }
    }

    private int simpleInvoke(int x, int y) {
        return local += x + y;
    }

    private void doublePlus(){
        double sum = 1;
        for (int i = 1; i <= round; i++) {
            sum += i;
        }
    }

    private void doubleMul(){
        double sum = 1;
        for (int i = 1; i <= round; i++) {
            sum *= i;
        }
    }

    private void doubleDiv(){
        double sum = 1;
        for (int i = 1; i <= round; i++) {
            sum /= i;
        }
    }


    int local = 0;


    public void invoke() {
        int sum = 1;
        for (int i = 1; i <= round; i++) {
            sum = simpleInvoke(sum, i);
        }
    }


    public void and() {
        int sum = 1;
        for (int i = 1; i <= round; i++) {
            sum &= i;
        }
    }
}


```

共测100次，取平均值。

下面是执行输出：

```
longMul finished in 109ms
doubleMul finished in 279ms
doublePlus finished in 284ms
mod finished in 919ms
plusMultiway finished in 38ms
mul finished in 106ms
subtract finished in 48ms
invoke finished in 149ms
choose finished in 187ms
plus finished in 50ms
empty finished in 46ms
longPlus finished in 80ms
div finished in 896ms
doubleDiv finished in 462ms
and finished in 47ms
```

empty是循环体带来的时间，所有时间减去empty的时间就是其方法体的时间。

可以发现在java中加减和位运算非常快。长整数会使得运算时间翻倍，但是依旧在可以接受的范围中。

函数调用和分支跳转更慢，但是依旧不会影响你的程序的总体时间。所以爱咋用就咋用。

浮点数的运算时间非常慢，比函数调用还慢。但是浮点数的除法却比整数除法快很多，可能是算法不同。

整数除法和取模异常慢，所以尽量避免。在取模运算的时候用long来存储累积和，只有在发生除法的时候才进行必要的取模。