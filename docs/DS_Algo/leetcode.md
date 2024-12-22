# leetcode

## 资料
* [滑动窗口与双指针](https://leetcode.cn/circle/discuss/0viNMK/)

## 定长滑动窗口

1. [定长子串中元音的最大数目](https://leetcode.cn/problems/maximum-number-of-vowels-in-a-substring-of-given-length/description/)：给你字符串 s 和整数 k 。请返回字符串 s 中长度为 k 的单个子字符串中可能包含的最大元音字母数。英文中的 元音字母 为（a, e, i, o, u）。示例：

    输入：s = "abciiidef", k = 3

    输出：3

    解释：子字符串 "iii" 包含 3 个元音字母。

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

2. [爱生气的书店老板](https://leetcode.cn/problems/grumpy-bookstore-owner/description/)：

    **关键思路：将问题拆分成两个子问题**

    ```python
    class Solution:
        def maxSatisfied(self, customers: List[int], grumpy: List[int], minutes: int) -> int:
            # s[0]: 老板不生气时的顾客数量之和
            # s[1]: 长度为 minutes 的连续子数组中，老板生气时的顾客数量之和，通过定长滑动窗口解决
            # 最终答案为 s0 + max_s1
            s = [0, 0]
            max_s1 = 0
            for i, (c, g) in enumerate(zip(customers, grumpy)):
                s[g] += c
                if i < minutes - 1:                     # 窗口长度不足 minutes
                    continue
                max_s1 = max(max_s1, s[1])              # 更新 max_s1
                if grumpy[i - minutes + 1]:             # 因为 s0 和 s1 是分开统计的，所以离开窗口时要判断
                    s[1] -= customers[i - minutes + 1]  # 窗口最左边元素离开窗口
            return s[0] + max_s1
    ```

3. [检查一个字符串是否包含所有长度为 K 的二进制子串](https://leetcode.cn/problems/check-if-a-string-contains-all-binary-codes-of-size-k/description/)：给你一个二进制字符串 s 和一个整数 k 。如果所有长度为 k 的二进制字符串都是 s 的子串，请返回 true ，否则请返回 false 。示例：
   
    输入：s = "00110110", k = 2

    输出：true

    解释：长度为 2 的二进制串包括 "00"，"01"，"10" 和 "11"。它们分别是 s 中下标为 0，1，3，2 开始的长度为 2 的子串。

    **关键思路：统计所有出现过的长度为 k 的 unique 子串，判断结果是否和 2 ** k 相等**

    ```python
    class Solution:
        def hasAllCodes(self, s: str, k: int) -> bool:
            ans = set()
            left = 0
            for right, c in enumerate(s):        # 1. 进入窗口
                if right < k - 1:                # 窗口大小不足
                    continue
                ans.add(s[left : right + 1])     # 2. 更新答案
                left += 1                        # 3. 离开窗口
            return len(ans) == 2 ** k
    ```

4. [几乎唯一子数组的最大和](https://leetcode.cn/problems/maximum-sum-of-almost-unique-subarray/description/)：

    **关键思路：利用 Counter 来统计每个数出现的次数**

    ```python
    class Solution:
        def maxSum(self, nums: List[int], m: int, k: int) -> int:
            ans = 0
            s = sum(nums[:k - 1])         # 先统计前 k-1 个数
            cnt = Counter(nums[:k - 1])
            for out, in_ in zip(nums, nums[k - 1:]):
                s += in_                  # 再添加一个数就是 k 个数了
                cnt[in_] += 1
                if len(cnt) >= m:
                    ans = max(ans, s)     # 是几乎唯一子数组，更新答案
                s -= out                  # 离开窗口
                cnt[out] -= 1
                if cnt[out] == 0:
                    del cnt[out]
            return ans
    ```

5. [可获得的最大点数](https://leetcode.cn/problems/maximum-points-you-can-obtain-from-cards/description/)：

    **关键思路：逆向思维 - 转换为求最小和 ➡️ 最小化剩下的点数和**

    ```python
    class Solution:
        def maxScore(self, cardPoints: List[int], k: int) -> int:
            n = len(cardPoints)
            m = n - k
            s = 0
            min_s = math.inf
            # 注意 python 的enumerate无法提前取到后面的元素，特殊情况的判断
            if (m == 0):
                return sum(cardPoints)
            for i, num in enumerate(cardPoints):
                s += num
                if i < m - 1:
                    continue
                min_s = min(min_s, s)
                s -= cardPoints[i - m + 1]
            return sum(cardPoints) - min_s
    ```

    更精简的写法：

    ```python
    class Solution:
        def maxScore(self, cardPoints: List[int], k: int) -> int:
            n = len(cardPoints)
            m = n - k
            min_s = s = sum(cardPoints[:m])
            for i in range(m, n):
                s += cardPoints[i] - cardPoints[i - m]
                min_s = min(min_s, s)
            return sum(cardPoints) - min_s
    ```