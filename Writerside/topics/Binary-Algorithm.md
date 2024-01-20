# Binary Algorithm

## 位运算符

`<<` signed

`<<<` unsigned

`>>` signed

`>>>` unsigned

`|` bitwise or

`&` bitwise and

`~` bitwise not

`^` bitwise xor

## XOR

Non-rounding addition

```
x ^ x = 0
x ^ 1s = ~x // 注： 1s = ~0
x ^ (~x) = 1s
c = a ^ b 👉 a ^ c = b, b ^ c = a // 交换两个数
a ^ b ^ c = a ^ (b ^ c) = (a ^ b) ^ c // associative
```

## 指定位置的位运算

- 将 x 最右边的 n 位清零：`x & (~0 << n)`
- 获取 x 的第 n 位值（0 或者 1）： `(x >> n) & 1`
- 获取 x 的第 n 位的幂值：`x & (1 << n)`
- 仅将第 n 位置为 1：`x | (1 << n)`
- 仅将第 n 位置为 0：`x & (~ (1 << n))`
- 将 x 最高位至第 n 位（含）清零：`x & ((1 << n) - 1)`

## 应用

判断奇偶：

- `x % 2 == 1` —> `(x & 1) == 1`
- `x % 2 == 0` —> `(x & 1) == 0`

`x >> 1` —> `x / 2`

- 即： `x = x / 2;` —> `x = x >> 1;`
- `mid = (left + right) / 2;` —>  `mid = (left + right) >> 1;`

`X = X & (X-1)` 清零最低位的 1

`X & -X` => 得到最低位的 1

`X & ~X` => 0

## LeetCode

- [https://leetcode-cn.com/problems/number-of-1-bits/](https://leetcode-cn.com/problems/number-of-1-bits/)
- [https://leetcode-cn.com/problems/power-of-two/](https://leetcode-cn.com/problems/power-of-two/)
- [https://leetcode-cn.com/problems/reverse-bits/](https://leetcode-cn.com/problems/reverse-bits/)
- [https://leetcode-cn.com/problems/n-queens/description/](https://leetcode-cn.com/problems/n-queens/description/)
- [https://leetcode-cn.com/problems/n-queens-ii/description/](https://leetcode-cn.com/problems/n-queens-ii/description/)