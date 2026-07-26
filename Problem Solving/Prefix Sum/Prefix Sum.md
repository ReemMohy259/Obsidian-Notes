
The **Prefix Sum Algorithm** is a preprocessing technique used to answer range sum queries efficiently. Instead of calculating the sum of elements repeatedly for every query, we build a new array called the **prefix sum array**, where each index stores the cumulative sum up to that position.

This reduces range sum query time from:
- **Brute Force:** `O(n)` 
- **Prefix Sum:** `O(1)

> [!NOTE] Note
> Prefix sum concept can be applied on any other operations rather than sum operation, but under **one condition:**
> - if this operation is **reversable** (e.g. sum, difference, division, multiplication, XOR)
> - Can't have prefix modulus because modulus operation didn't has a reverse operation

---
# The Problem

Suppose we have an array:

```txt
arr = [2, 4, 1, 7, 3, 6]
```

We want to answer multiple queries like:

```txt
Sum from index L to R
```

Example:

```txt
Sum(1,4) = 4 + 1 + 7 + 3 = 15
```

---

# Brute Force Solution

For every query:

```cpp
int sum = 0;
for(int i = L; i <= R; i++) {
    sum += arr[i];
}
```

### Complexity

- Each query takes `O(n)`
- If there are `q` queries:

```txt
Total Complexity = O(q * n)
```

This becomes slow when:
- `n` is large
- `q` is large
---
# Prefix Sum Idea

Instead of recomputing sums repeatedly. We preprocess cumulative sums once.

```txt
prefix[i] = sum of elements from 0 to i
```

---
# Building the Prefix Array

For:

```txt
arr = [2, 4, 1, 7, 3, 6]
```

We build:

|Index|arr[i]|prefix[i]|
|---|---|---|
|0|2|2|
|1|4|6|
|2|1|7|
|3|7|14|
|4|3|17|
|5|6|23|

---
# Formula

The prefix sum relation is: `prefix[i]=prefix[i-1]+arr[i]`

---
# Range Sum Query

To calculate:

```txt
Sum(L, R)
```

We use:

- Total sum until `R`
- Remove sum before `L`
Formula: `sum(L,R)=prefix[R]-prefix[L-1]`

**Special case:**

```txt
If L == 0:
sum = prefix[R]
```

---
# Example Walkthrough

Array:

```txt
arr = [2, 4, 1, 7, 3, 6]
```

Prefix:

```txt
prefix = [2, 6, 7, 14, 17, 23]
```

Find:

```txt
Sum(1,4)
```

Using formula:

```txt
prefix[4] - prefix[0]
= 17 - 2
= 15
```

Correct answer:

```txt
4 + 1 + 7 + 3 = 15
```

---
# Implementation

```java
import java.util.*;

public class Main {

    public static void main(String[] args) {

        int[] arr = {2, 4, 1, 7, 3, 6};

        int n = arr.length;

        int[] prefix = new int[n];

        // Build prefix sum
        prefix[0] = arr[0];

        for (int i = 1; i < n; i++) {
            prefix[i] = prefix[i - 1] + arr[i];
        }

        // Query
        int L = 1;
        int R = 4;

        int sum;

        if (L == 0)
            sum = prefix[R];
        else
            sum = prefix[R] - prefix[L - 1];

        System.out.println(sum);
    }
}
```

---
# Complexity Analysis

### Building Prefix Array

```txt
O(n)
```

### Each Query

```txt
O(1)
```

### Total

```txt
O(n + q)
```

This is much faster than:

```txt
O(q * n)
```

---
# Famous Examples 

### 1. Range Sum Query
**The Problem:** You are given an array of numbers. You have to answer many questions asking for the sum of elements between two indices, `left` and `right`.

### 2. Subarray Sum Equals K

Now for a trickier problem. This one shows the true power of prefix sums when combined with another tool: the **Hash Map**.

**The Problem:** Given an array and a target number `k`, find the total number of continuous subarrays whose sum equals `k`.

**Example:** `nums = [3, 4, 7, 2, -3, 1, 4, 2]`, `k = 7`

The subarrays that sum to 7 are `[3, 4]`, `[7]`, `[7, 2, -3, 1]`, and `[1, 4, 2]`. The answer is **4**.

**The Logic:**
We know that the sum of a subarray from index `i` to `j` is `prefix[j] - prefix[i-1]`. So, we are looking for cases where: `prefix[j] - prefix[i-1] = k`.

Let’s rearrange that equation: `prefix[i-1] = prefix[j] - k`.

This is our key insight! While we are calculating our prefix sums one by one (let’s call the current one `current_sum`), we can simply check: **"Have we seen a previous prefix sum that equals** `**current_sum - k**` ?"

If we have, it means a subarray ending at our current position sums to `k`!

A hash map is perfect for this. We can use it to store the prefix sums we’ve already seen and how many times we’ve seen them.

**The Algorithm:**

- Initialize a hash map `seen_sums` with `{0: 1}`. This is a crucial starting point to handle subarrays that start from the very beginning of the array.
- Initialize `count = 0` and `current_sum = 0`.
- Loop through the array’s numbers:
- Add the number to `current_sum`.
- Look in our hash map for `current_sum - k`. If we find it, we add the number of times it appeared to our `count`.
- Update our hash map by adding `current_sum` to it.
```java
import java.util.HashMap;

public class Main {

    public static int subarraySum(int[] nums, int k) {

        // {prefixSum : frequency}
        HashMap<Integer, Integer> seenSums = new HashMap<>();

        // Base case
        seenSums.put(0, 1);

        int count = 0;
        int currentSum = 0;

        for (int num : nums) {

            // Build prefix sum
            currentSum += num;

            // Check if (currentSum - k) exists
            if (seenSums.containsKey(currentSum - k)) {
                count += seenSums.get(currentSum - k);
            }

            // Store current prefix sum
            seenSums.put(
                currentSum,
                seenSums.getOrDefault(currentSum, 0) + 1
            );
        }

        return count;
    }

    public static void main(String[] args) {

        int[] nums = {3, 4, 7, 2, -3, 1, 4, 2};
        int k = 7;

        System.out.println(subarraySum(nums, k));
    }
}
```