# HW

## 使用 ok 自动化测试工具

```bash
cs61a/lab/lab00$ python ok --local -u
```

## HW01: Functions, Control

1. a_plus_abs_b

    ```python
    from operator import add, sub

    def a_plus_abs_b(a, b):
        """Return a+abs(b), but without calling abs.

        >>> a_plus_abs_b(2, 3)
        5
        >>> a_plus_abs_b(2, -3)
        5
        >>> a_plus_abs_b(-1, 4)
        3
        >>> a_plus_abs_b(-1, -4)
        3
        """
        if b < 0:
            f = sub
        else:
            f = add
        return f(a, b)
    ```

2. two_of_three

    ```python
    def two_of_three(i, j, k):
        """Return m*m + n*n, where m and n are the two smallest members of the
        positive numbers i, j, and k.

        >>> two_of_three(1, 2, 3)
        5
        >>> two_of_three(5, 3, 1)
        10
        >>> two_of_three(10, 2, 8)
        68
        >>> two_of_three(5, 5, 5)
        50
        """
        return i ** 2 + j ** 2 + k ** 2 - max(i, j, k) ** 2
    ```

3. largest_factor

    ```python
    def largest_factor(n):
        """Return the largest factor of n that is smaller than n.

        >>> largest_factor(15) # factors are 1, 3, 5
        5
        >>> largest_factor(80) # factors are 1, 2, 4, 5, 8, 10, 16, 20, 40
        40
        >>> largest_factor(13) # factors are 1, 13
        1
        """
        for i in range(n // 2, 0, -1):
            if n % i == 0:
                return i
    ```

4. hailstone

    ```python   
    def hailstone(n):
        """Print the hailstone sequence starting at n and return its
        length.

        >>> a = hailstone(10)
        10
        5
        16
        8
        4
        2
        1
        >>> a
        7
        >>> b = hailstone(1)
        1
        >>> b
        1
        """
        step = 1
        while n != 1:
            print(n)
            if n % 2 == 0:
                n //= 2
            else:
                n = 3 * n + 1
            step += 1
        print(1)
        return step
    ```