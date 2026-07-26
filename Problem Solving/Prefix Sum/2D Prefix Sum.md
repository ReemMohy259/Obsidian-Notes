# Definition

A **2D Prefix Sum** (also called a **Prefix Sum Matrix**) is an extension of the 1D prefix sum technique.

It is used to efficiently calculate:

```txt
Sum of elements inside any submatrix
```

Instead of recalculating sums repeatedly, we preprocess the matrix once.

After preprocessing:

```txt
Each query can be answered in O(1)
```

---
# The Problem

Suppose we have a matrix:

```txt
1   2   3   4
5   6   7   8
9  10  11  12
13 14  15  16
```

We want to answer queries like:

```txt
Find the sum of elements inside a rectangle
```

Example:

```txt
From (1,1) to (2,2)
```

which means:

```txt
6  7
10 11
```

Answer:

```txt
6 + 7 + 10 + 11 = 34
```

---

# Brute Force Approach

For every query:

```txt
Loop through all rows and columns inside the rectangle
```

Complexity:

```txt
O(rows × cols) per query
```

Very slow for many queries.

---
# Core Idea of 2D Prefix Sum

We preprocess cumulative sums.

Each cell stores:

```txt
Sum of all elements from (0,0) → (i,j)
```

---
# Prefix Matrix Definition

Let:

```txt
prefix[i][j]
```

represent:

```txt
Sum of rectangle from top-left corner (0,0)
to current cell (i,j)
```

---
# Construction Formula

The key formula is:
`prefix[i][j]=arr[i][j]+prefix[i-1][j]+prefix[i][j-1]-prefix[i-1][j-1]`
##### Why Do We Subtract?
Notice:
```txt
prefix[i-1][j]
```
and
```txt
prefix[i][j-1]
```
both include:
```txt
prefix[i-1][j-1]
```
So the overlap gets counted twice.
We subtract it once.
![[Pasted image 20260512233010.png|615]]

##### Visualization
Suppose we calculate:

```txt
prefix[2][2]
```

We add:
- Top rectangle
- Left rectangle
- Current value
    

But:
```txt
Top-left rectangle overlaps
```

So we subtract it.

---
# Java Code — Building 2D Prefix Sum

```java
int[][] arr = {
    {1, 2, 3, 4},
    {5, 6, 7, 8},
    {9, 10, 11, 12},
    {13, 14, 15, 16}
};

int n = arr.length;
int m = arr[0].length;

int[][] prefix = new int[n][m];

for (int i = 0; i < n; i++) {

    for (int j = 0; j < m; j++) {

        prefix[i][j] = arr[i][j];

        // top
        if (i > 0)
            prefix[i][j] += prefix[i - 1][j];

        // left
        if (j > 0)
            prefix[i][j] += prefix[i][j - 1];

        // remove overlap
        if (i > 0 && j > 0)
            prefix[i][j] -= prefix[i - 1][j - 1];
    }
}
```

---
# Querying Submatrix Sum

Suppose we want:
```txt
Sum from (r1,c1) → (r2,c2)
```

Formula:
`sum = prefix[r2][c2] - prefix[r1-1][c2] - prefix[r2][c1-1] + prefix[r1-1][c1-1]`

---
# Inclusion-Exclusion Principle

We start with:

```txt
Entire rectangle until (r2,c2)
```

Then remove:
- Extra top area
- Extra left area
But:
```txt
Top-left area gets removed twice
```
**So we add it back once.**

---
# Visual Explanation

To get:

```txt
(1,1) → (2,2)
```

We do:

```txt
Total rectangle
- top strip
- left strip
+ overlap
```

---
# Example Query

Matrix:

```txt
1   2   3   4
5   6   7   8
9  10  11  12
13 14  15  16
```

Find:

```txt
Sum from (1,1) → (2,2)
```

Submatrix:

```txt
6   7
10  11
```

Expected:

```txt
34
```

---
# Using Formula

From prefix matrix:

```txt
prefix[2][2] = 54
prefix[0][2] = 6
prefix[2][0] = 15
prefix[0][0] = 1
```

Compute:

```txt
54 - 6 - 15 + 1
= 34
```
---
# Java Query Code

```java
int total = prefix[r2][c2];

if (r1 > 0)
    total -= prefix[r1 - 1][c2];

if (c1 > 0)
    total -= prefix[r2][c1 - 1];

if (r1 > 0 && c1 > 0)
    total += prefix[r1 - 1][c1 - 1];
```

---

# Full Java Implementation

```java
import java.util.*;

public class Main {

    public static void main(String[] args) {

        int[][] arr = {
            {1, 2, 3, 4},
            {5, 6, 7, 8},
            {9, 10, 11, 12},
            {13, 14, 15, 16}
        };

        int n = arr.length;
        int m = arr[0].length;

        int[][] prefix = new int[n][m];

        // Build prefix matrix
        for (int i = 0; i < n; i++) {

            for (int j = 0; j < m; j++) {

                prefix[i][j] = arr[i][j];

                if (i > 0)
                    prefix[i][j] += prefix[i - 1][j];

                if (j > 0)
                    prefix[i][j] += prefix[i][j - 1];

                if (i > 0 && j > 0)
                    prefix[i][j] -= prefix[i - 1][j - 1];
            }
        }

        // Query
        int r1 = 1, c1 = 1;
        int r2 = 2, c2 = 2;

        int total = prefix[r2][c2];

        if (r1 > 0)
            total -= prefix[r1 - 1][c2];

        if (c1 > 0)
            total -= prefix[r2][c1 - 1];

        if (r1 > 0 && c1 > 0)
            total += prefix[r1 - 1][c1 - 1];

        System.out.println(total);
    }
}
```

---
# Complexity Analysis

### Building Prefix Matrix

```txt
O(n × m)
```

---

### Each Query

```txt
O(1)
```

---

### Total

```txt
O(n × m + q)
```

where:
- `n` = rows
- `m` = columns
- `q` = number of queries

---
# Space Complexity

```txt
O(n × m)
```

for storing the prefix matrix.
