# leetcode

## 资料
* [滑动窗口与双指针](https://leetcode.cn/circle/discuss/0viNMK/)

## 定长滑动窗口

1. [定长子串中元音的最大数目](https://leetcode.cn/problems/maximum-number-of-vowels-in-a-substring-of-given-length/description/)：给你字符串 s 和整数 k 。请返回字符串 s 中长度为 k 的单个子字符串中可能包含的最大元音字母数。英文中的 元音字母 为（a, e, i, o, u）。示例：

    输入：s = "abciiidef", k = 3
    输出：3
    解释：子字符串 "iii" 包含 3 个元音字母。

    解答：

    ```python
    class Solution:
        def maxVowels(self, s: str, k: int) -> int:
            ans = vowel = 0
            for i, c in enumerate(s):
                if c in "aeiou":            # 1. 进入窗口
                    vowel += 1
                if i < k - 1:               # 窗口大小不足 k
                    continue
                ans = max(ans, vowel)       # 2. 更新答案
                if s[i - k + 1] in "aeiou": # 3. 离开窗口
                    vowel -= 1
            return ans
    ```