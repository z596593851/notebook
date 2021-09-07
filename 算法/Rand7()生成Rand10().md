### 题目：
已有方法 rand7 可生成 1 到 7 范围内的均匀随机整数，试写一个方法 rand10 生成 1 到 10 范围内的均匀随机整数。
### 解析：
如果我们存在一个 `randK` 的函数，对其执行 n 次，我们能够等概率产生 [0,K^n−1] 范围内的数值。

所以我们生成两次 rand7-1 ，来组成一个7进制的数，可以表示的范围是 [0,48] ，然后我们取其中的 [1,10] 即可。

```java
class Solution extends SolBase {
    public int rand10() {
        while (true) {
            int ans = (rand7() - 1) * 7 + (rand7() - 1); // 进制转换
            if (1 <= ans && ans <= 10) return ans;
        }
    }
}
```
我们发现，在上述解法中，范围 [0,48] 中，只有 [1,10] 范围内的数据会被接受返回，其余情况均被拒绝重试。

为了尽可能少的调用 `rand7` 方法，我们可以从 [0,48] 中取与 [1,10] 成倍数关系的数，来进行转换。

我们可以取 [0,48] 中的 [1,40] 范围内的数来代指 [1,10]。
```java
class Solution extends SolBase {
    public int rand10() {
        while (true) {
            int ans = (rand7() - 1) * 7 + (rand7() - 1); // 进制转换
            if (1 <= ans && ans <= 40) return ans % 10 + 1;
        }
    }
}
```