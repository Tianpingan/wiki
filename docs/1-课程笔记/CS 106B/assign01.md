链接：https://web.stanford.edu/class/archive/cs/cs106b/cs106b.1228/assignments/1-cpp/

## Perfect numbers
完美数的定义：是一个值等于其真因数之和的数。

> 真因数是指一个自然数除自身以外的因数

关键函数`divisorSum(n)`：求得`n`的真因数之和。
```C++
long divisorSum(long n) {
    long total = 0;
    for (long divisor = 1; divisor < n; divisor++) {
        if (n % divisor == 0) {
            total += divisor;
        }
    }
    return total;
}
```
就是从`1`到`n - 1`进行枚举，如果其能被`n`整除，说明它是`n`的真因数，将其值加到`total`上面。

接下来我们启动程序，在弹出的命令行中选择需要测试的`perfect.cpp`文件。
![[Pasted image 20220828220004.png]]
此`main`搜索从 1 到 40000 的完美数。
接下来能看到测试结果：
![[Pasted image 20220828220049.png]]

**Q1**：Roughly how long did it take your computer to do the search? How many perfect numbers were found and what were they?
**answer**：根据结果可知，搜索`1`到`40000`的完美数，我的电脑需要`2.043`秒。
![[Pasted image 20220828220236.png]]
找到了4个完美数，如下：
![[Pasted image 20220828220402.png]]
分别是`6`、`28`、`496`、`8128`。

**SimpleTest的教程：**
就是这个课程提供了个简单的单元测试框架，我们可以使用这个框架来测试我们的代码。

进行测试时，测试用例需要被包含在宏`PROVIDED_TEST`或`STUDENT_TEST`中，这两种效果一样，只是用来区分是不是学生自己编写的测试。

常用的测试宏包括：
- `EXPECT_EQUAL`
	- `EXPECT_EQUAL(reversed("but"), "tub");`
- `EXPECT`
	- `EXPECT(isPalindrome("racecar"));`
	- `EXPECT(x > y && y != z || y == 0);`
- `EXPECT_ERROR`
	- 如果其中函数调用了`error()`，产生错误，则通过测试。
- `EXPECT_NO_ERROR`
	- 与上者相反
- `TIME_OPERATION`
	- `TIME_OPERATION(v.size(), v.sort());`
	- 统计函数调用时间


接着，通过反复实验，我的计算机可以在大约一分钟内完成的最大搜索大小是`24000`左右。
**Q2**. Record the timing results for `findPerfects` that you observed into a table. (old-school table with text-based rows and columns is just fine!)
编写的测试代码如下：
```C++
STUDENT_TEST("My Test"){
    TIME_OPERATION(30000, findPerfects(30000));
    TIME_OPERATION(60000, findPerfects(60000));
    TIME_OPERATION(120000, findPerfects(120000));
    TIME_OPERATION(240000, findPerfects(240000));
}
```
计时结果如下：
![[Pasted image 20220828221800.png]]
绘制表格如下：

略

**Q3**. Does it take the same amount of work to compute `isPerfect` on the number 10 as it does on the number 1000? Why or why not? Does it take the same amount of work for `findPerfects` to search the range of numbers from 1-1000 as it does to search the numbers from 1000-2000? Why or why not?
`isPerfect`计算数字 10 和计算数字 1000 所需的工作量是不同的，
`findPerfects`计算 1-1000 范围和1000-2000的工作量也是不同的。
（`divisorSum`计算量不同）


**Q4**. Extrapolate from the data you gathered and make a prediction: how long will it take `findPerfects` to reach the fifth perfect number?
第5个大约 3300 万左右，估算要xx秒
$f(x) = k\times x^2$
$f(240000) = k \times 240000^2 = 67, k = \frac{67}{240000^2}$
所以
$f(3300000) = \frac{67}{240000^2} \times 3300000^2 = 12,667.1875$ ，大约两百多分钟。

**Q5**. Do any of the tests still pass even with this broken function? Why or why not?
`isPerfect(n)`处理负数正确是因为其中的判定条件`n == divisorSum(n)`永远是错误的，因为后者`divisorSum(n)`在`n`小于`0`时，结果永为`0`。
我们修改`total=1`后，依旧有测试通过，如下：
![[Pasted image 20220829064647.png]]
最后两个测试正确的原因如上，其中的判定条件`n == divisorSum(n)`永远是错误的，因为后者`divisorSum(n)`在`n`小于`0`时永远为`1`（`total`的初始值）。
倒数第三个测试只是测时间。
第三个测试，设初始值`total=1`时，`12`和`98765`依旧满足不了`n == divisorSum(n)`，所以测试正确。


每次我们找到一个真因数时，实际上只需要检查到`sqrt(n)`。
因为`n`的真因数可以分成两类，一类是小于`sqrt(n)`的，一类是不小于`sqrt(n)`的，而且这二者存在一一对应的关系。
**smarterSum**：
```C++
long smarterSum(long n) {
    /* TODO: Fill in this function. */
    long total = 0;
    for (long divisor = 1; divisor <= sqrt(n); divisor++) {
        if (n % divisor == 0) {
            if(divisor != n)
                total += divisor;
            if(n / divisor == divisor || n / divisor == n)
                continue;
            total += n / divisor;
        }
    }
    return total;
}
```

**Q6**. Describe the testing strategy you used for your test cases to confirm `smarterSum` is working correctly.
```C++
STUDENT_TEST("My Test for smarterSum"){
    EXPECT_EQUAL(smarterSum(25), divisorSum(25));
    EXPECT_EQUAL(smarterSum(100), divisorSum(100));
    EXPECT_EQUAL(smarterSum(0), divisorSum(0));
    EXPECT_EQUAL(smarterSum(1), divisorSum(1));
    EXPECT_EQUAL(smarterSum(-1), divisorSum(-1));
}
```

**Q7**. Record your timing results for `findPerfectsSmarter` into a table.



**Q8**. Make a prediction: how long will `findPerfectsSmarter` take to reach the fifth perfect number?




```C++
long findNthPerfectEuclid(long n) {
    /* TODO: Fill in this function. */
//    return 0;

    int k = 1;
    long value = 0;
    while(n--)
    {
        while(divisorSum((1ll << k) - 1) != 1)
        {
            k++;
        }
        value = (1ll << (k - 1)) * ((1ll << k) - 1);
        k++;
    }
    return value;
}
```

**Q9**. Explain how you chose your specific test cases and why they lead you to be confident `findNthPerfectEuclid` is working correctly.

##  Soundex Search
The Soundex algorithm operates on an input surname and converts the name into its Soundex code.
输入一个姓氏，输出一个四字符的编码。
步骤：
- 丢弃姓氏中的所有非字母字符
- 使用下表将每个字母编码为数字。
	- ![[Pasted image 20220829102427.png]]
- 合并相邻的重复数字
- 将编码的第一个数字替换为姓氏中的第一个字母
- 从删除中删除所有零
- 通过用零填充或截断多余部分，使编码的长度正好为 4。

**Q10**. What is the Soundex code for "Angelou"? What is the code for your own surname?


**Q11**. Before writing any code, brainstorm your plan of attack and sketch how you might decompose the work into smaller tasks. Briefly describe your decomposition strategy.

```C++
string removeNonLetters(string s) {
    string result = charToString(s[0]);
    for (int i = 1; i < s.length(); i++) {
        if (isalpha(s[i])) {
            result += s[i];
        }
    }
    return result;
}
```
存在的bug：可能第一个字符就是符号。

