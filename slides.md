---
theme: ./theme
background: rgb(0, 0, 0, 0.4)
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
css: unocss
title: Cachelab
---

# Cachelab

Zhuang Jiayi, Nov 2022.

---

# Part A: Cache Simulator

Build a cache simulator to evaluate the performance of a cache memory.

That is, the first time we build a **CLI app** from scratch.

## What is CLI(Command-Line Interface) app?

- Write programs that do one thing and do it well.
- Write programs to work together.
- Write programs to handle text streams, because that is a universal interface.

<div align="right">from <i>Unix Philosophy</i></div>

---

## Parse command line arguments

Use `getopt` in `unistd.h`.

Get help from `man 3 getopt`.

```c {all} {maxHeight: '300px'}
/// Verbose flag.
int verbose = 0;
/// Set index size (2^s).
int set_size = -1;
/// Number of lines per set
int associativity = -1;
/// Block offset size (2^b).
int block_size = -1;
/// Trace file path.
char *trace_file_path = 0;

while ((opt = getopt(argc, argv, "hvs:E:b:t:")) != -1) {
    switch (opt) {
    case 'h':
      usage(0);
    case 'v':
      verbose = 1;
      break;
    case 's':
      set_size = atoi(optarg);
      break;
    case 'E':
      associativity = atoi(optarg);
      break;
    case 'b':
      block_size = atoi(optarg);
      break;
    case 't':
      trace_file_path = optarg;
      break;
    case '?':
      usage(1);
    }
  }

if (set_size < 0 || associativity < 0 || block_size < 0 || !trace_file_path) {
  BAIL(1, "Missing required command line argument\n");
}
```

---

## Robustness

Check the validity of command line arguments:

- Additonal arguments: ignore or reject?
- Missing arguments: how to detect?
- Illegal arguments: how to detect?

Pay attention to side effects of `getopt`:

- `argv` is modified to split the arguments.
- No memory is allocated.

---

## LRU implementation

<p></p>

### **1. Naive but direct way**

Generate a counter/timestamp for each operation.
Find and delete the biggest one when eviction happens.

**Faults:** Racing when generating the timestamp; counter overflow; **cache-unfriendly**.

### **2. Better way**

Each operation moves the block to the head of the list.
Delete the tail when eviction happens.

**Cache-friendly:** Recently used blocks are gathered at the head of the list.

### **3. Linked list optimization**

- Use a **doubly linked list** to avoid copying when moving a node.
- Use a **hash table** to avoid traversing the whole list when finding a node.

Read: $O(1)$, Write: $O(1)$, Eviction: $O(1)$.

*A bit more complex, so I didn't implement it* :D

---
layout: two-col
---

# Part B: Matrix Transpose

Write an algorithm to transpose a $M \times N$ matrix, in cache-friendly way.

## **v0.0.0:** baseline

::left::

```c {all} {maxHeight: '300px'}
/*
 * trans - A simple baseline transpose function, not optimized for the cache.
 */
char trans_desc[] = "Simple row-wise scan transpose";
void trans(int M, int N, int A[N][M], int B[M][N]) {
  int i, j, tmp;

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (i = 0; i < N; i++) {
    for (j = 0; j < M; j++) {
      tmp = A[i][j];
      B[j][i] = tmp;
    }
  }

  ENSURES(is_transpose(M, N, A, B));
}
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 |  868  |  1180  | <MissRate :hits="868" :misses="1180"/>  |
| 64×64 | 3472  |  4720  | <MissRate :hits="3472" :misses="4720"/> |
| 60×68 | 3846  |  4314  | <MissRate :hits="3846" :misses="4314"/> |

---
layout: two-col
---

## **v1.1.0:** 8×1 loop unrolling

You can always trust loop unrolling, unless you have a good reason not to.

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_8_way - 8-way loop unrolling.
 */
char trans_8_way_desc[] = "8-way loop unrolling";
void trans_8_way(int M, int N, int A[N][M], int B[M][N]) {
  int i, j;
  int t0, t1, t2, t3, t4, t5, t6, t7;

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (i = 0; i < N; i++) {
    for (j = 0; j + 7 < M; j += 8) {
      t0 = A[i][j + 0];
      t1 = A[i][j + 1];
      t2 = A[i][j + 2];
      t3 = A[i][j + 3];
      t4 = A[i][j + 4];
      t5 = A[i][j + 5];
      t6 = A[i][j + 6];
      t7 = A[i][j + 7];

      B[j + 0][i] = t0;
      B[j + 1][i] = t1;
      B[j + 2][i] = t2;
      B[j + 3][i] = t3;
      B[j + 4][i] = t4;
      B[j + 5][i] = t5;
      B[j + 6][i] = t6;
      B[j + 7][i] = t7;
    }
    for (; j < M; j++)
      B[j][i] = A[i][j];
  }

  ENSURES(is_transpose(M, N, A, B));
}
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 |  896  |  1152  | <MissRate :hits="896" :misses="1152"/>  |
| 64×64 | 3584  |  4608  | <MissRate :hits="3584" :misses="4608"/> |
| 60×68 | 3882  |  4278  | <MissRate :hits="3882" :misses="4278"/> |

Score: 0.0 + 0.0 + 0.0 = 0.0

---
layout: two-col
---

## **v1.2.0:** 4×4 blocked transpose, row first

The answer is ~~42~~ in *writeup*.

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_row - 4*4 blocked transpose, row first.
 * Size 4 is the best.
 *
 * Score: 3.1 + 1.3 + 7.7 = 12.1
 */
char trans_blocked_row_desc[] = "4*4 blocked transpose, row first";
#define BSIZE 4
void trans_blocked_row(int M, int N, int A[N][M], int B[M][N]) {
  int i, j, ii, jj;
  int en = BSIZE * ((N - 1) / BSIZE); /* block count at dimension N */
  int em = BSIZE * ((M - 1) / BSIZE); /* block count at dimension M */

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (ii = 0; ii <= en; ii += BSIZE)
    for (jj = 0; jj <= em; jj += BSIZE)
      for (i = ii; i < ii + BSIZE && i < N; i++)
        for (j = jj; j < jj + BSIZE && j < M; j++)
          B[j][i] = A[i][j];

  ENSURES(is_transpose(M, N, A, B));
}
#undef BSIZE
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 | 1564  |  484   | <MissRate :hits="1564" :misses="484"/>  |
| 64×64 | 6304  |  1888  | <MissRate :hits="6304" :misses="1888"/> |
| 60×68 | 6331  |  1829  | <MissRate :hits="6331" :misses="1829"/> |

Score: 3.1 + 1.3 + 7.7 = 12.1

---
layout: two-col
---

## **v1.3.0:** 1×4 blocked transpose, column first

It did. *But I don't know why.*

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_col - 1*4 blocked transpose, col first.
 *
 * Score: 3.7 + 1.8 + 8.0 = 13.5
 */
char trans_blocked_col_desc[] = "1*4 blocked transpose, col first";
#define BSIZE 4
void trans_blocked_col(int M, int N, int A[N][M], int B[M][N]) {
  int i, j, jj;
  int em = BSIZE * ((M - 1) / BSIZE); /* block count at dimension M */

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (jj = 0; jj <= em; jj += BSIZE)
    for (i = 0; i < N; i++)
      for (j = jj; j < jj + BSIZE && j < M; j++)
        B[j][i] = A[i][j];

  ENSURES(is_transpose(M, N, A, B));
}
#undef BSIZE
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 | 1588  |  460   | <MissRate :hits="1588" :misses="460"/>  |
| 64×64 | 6352  |  1840  | <MissRate :hits="6352" :misses="1840"/> |
| 60×68 | 6358  |  1802  | <MissRate :hits="6358" :misses="1802"/> |

Score: 3.7 + 1.8 + 8.0 = 13.5

---
layout: two-col
---

## **v1.4.0:** 1×4 blocked along with 4×1 loop unrolling

Today we were able to put it together, we just have to keep it going.

::left::
  
```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_4_way - 1*4 blocked transpose along with 4-way loop unrolling.
 *
 * Score: 5.0 + 4.0 + 9.6 = 18.6
 */
char trans_blocked_4_way_desc[] =
    "1*4 blocked transpose along with 4-way loop unrolling";
void trans_blocked_4_way(int M, int N, int A[N][M], int B[M][N]) {
  int i, j, jj;
  int t0, t1, t2, t3;
  int em = 4 * ((M - 1) / 4); /* block count at dimension M */

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (jj = 0; jj <= em; jj += 4)
    for (i = 0; i < N; i++) {
      if (jj + 4 <= M) {
        t0 = A[i][jj + 0];
        t1 = A[i][jj + 1];
        t2 = A[i][jj + 2];
        t3 = A[i][jj + 3];

        B[jj + 0][i] = t0;
        B[jj + 1][i] = t1;
        B[jj + 2][i] = t2;
        B[jj + 3][i] = t3;
      } else
        for (j = jj; j < M; j++)
          B[j][i] = A[i][j];
    }

  ENSURES(is_transpose(M, N, A, B));
}
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 | 1636  |  412   | <MissRate :hits="1636" :misses="412"/>  |
| 64×64 | 6544  |  1648  | <MissRate :hits="6544" :misses="1648"/> |
| 60×68 | 6522  |  1638  | <MissRate :hits="6522" :misses="1638"/> |

Score: 5.0 + 4.0 + 9.6 = 18.6

---
layout: two-col
---

## **v1.5.0:** 1×8 blocked along with 8×1 loop unrolling

Keep *calm* and *analyze*...

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_8_way - 1*8 blocked transpose along with 8-way loop unrolling.
 *
 * Score: 8.0 + 0.0 + 10.0 = 18.0
 */
char trans_blocked_8_way_desc[] =
    "1*8 blocked transpose along with 8-way loop unrolling";
void trans_blocked_8_way(int M, int N, int A[N][M], int B[M][N]) {
  int i, j, jj;
  int t0, t1, t2, t3, t4, t5, t6, t7;

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (jj = 0; jj <= 8 * ((M - 1) / 8); jj += 8)
    for (i = 0; i < N; i++) {
      if (jj + 8 <= M) {
        t0 = A[i][jj + 0];
        t1 = A[i][jj + 1];
        t2 = A[i][jj + 2];
        t3 = A[i][jj + 3];
        t4 = A[i][jj + 4];
        t5 = A[i][jj + 5];
        t6 = A[i][jj + 6];
        t7 = A[i][jj + 7];

        B[jj + 0][i] = t0;
        B[jj + 1][i] = t1;
        B[jj + 2][i] = t2;
        B[jj + 3][i] = t3;
        B[jj + 4][i] = t4;
        B[jj + 5][i] = t5;
        B[jj + 6][i] = t6;
        B[jj + 7][i] = t7;
      } else
        for (j = jj; j < M; j++)
          B[j][i] = A[i][j];
    }

  ENSURES(is_transpose(M, N, A, B));
}
```

::right::

$$
\frac{\mathtt{sizeof}(\mathrm{Cacheline})}{\mathtt{sizeof}(\mathtt{int})}=\frac{2^5}{4}=8
$$

|       | Hits  | Misses |                      Miss Rate                      |
| :---: | :---: | :----: | :-------------------------------------------------: |
| 32×32 | 1764  |  284   |       <MissRate :hits="1764" :misses="284"/>        |
| 64×64 | 3584  |  4608  | <MissRate color="red" :hits="3584" :misses="4608"/> |
| 60×68 | 6658  |  1502  |       <MissRate :hits="6658" :misses="1502"/>       |

Score: 8.0 + <span color="red">0.0</span> + 10.0 = 18.0

---

### What happened?

We have a cache miss rate **over 50%** for 64×64 matrix. We have to find out why.

Modify `csim.c` to print out which cache line is being accessed in verbose mode.

<div class="grid grid-cols-6 gap-4">

<div class="col-span-4">
```text {all} {maxHeight: '200px'}
Set 0,  Block 0   L 8888800000,4 miss
Set 0,  Block 4   L 8888800004,4 hit
Set 0,  Block 8   L 8888800008,4 hit
Set 0,  Block 12  L 888880000c,4 hit
Set 0,  Block 16  L 8888800010,4 hit
Set 0,  Block 20  L 8888800014,4 hit
Set 0,  Block 24  L 8888800018,4 hit
Set 0,  Block 28  L 888880001c,4 hit
Set 0,  Block 0   S 8888840000,4 miss eviction
Set 8,  Block 0   S 8888840100,4 miss
Set 16, Block 0   S 8888840200,4 miss
Set 24, Block 0   S 8888840300,4 miss
Set 0,  Block 0   S 8888840400,4 miss eviction
Set 8,  Block 0   S 8888840500,4 miss eviction
Set 16, Block 0   S 8888840600,4 miss eviction
Set 24, Block 0   S 8888840700,4 miss eviction
Set 8,  Block 0   L 8888800100,4 miss eviction
Set 8,  Block 4   L 8888800104,4 hit
Set 8,  Block 8   L 8888800108,4 hit
Set 8,  Block 12  L 888880010c,4 hit
Set 8,  Block 16  L 8888800110,4 hit
Set 8,  Block 20  L 8888800114,4 hit
Set 8,  Block 24  L 8888800118,4 hit
Set 8,  Block 28  L 888880011c,4 hit
Set 0,  Block 4   S 8888840004,4 miss eviction
Set 8,  Block 4   S 8888840104,4 miss eviction
Set 16, Block 4   S 8888840204,4 miss eviction
Set 24, Block 4   S 8888840304,4 miss eviction
Set 0,  Block 4   S 8888840404,4 miss eviction
Set 8,  Block 4   S 8888840504,4 miss eviction
......
```

- `A[0][0]` and `B[0][0]` are mapped into a same line.
- `B[0][0]` and `B[4][0]` are even mapped into a same line!

</div>

<div class="col-span-1">

A[0:8, 56:63]

<CacheBlock :lines="[7,15,23,31,7,15,23,31]"/>

</div>

<div class="col-span-1">

B[56:63, 0:8]

<CacheBlock :lines="[0,8,16,24,0,8,16,24]"/>

</div>

</div>

When we read the lower part of a block in A,
the upper part of the block is evicted from the cache,
so we have to load it again.
That's why we have such a high cache miss rate.

---

### Solution

- Miss penalty is 10 times or even more much higher than hit cost.
- We can reduce the number of misses by performing more memory accesses.
**It worths**.

Here is a way to transpose a block of 8×8 without accessing the upper part and
the lower part of a block at the same time.

*See: [https://zhuanlan.zhihu.com/p/387662272](https://zhuanlan.zhihu.com/p/387662272)*

<div class="relative">
<img src="/1-0.webp" class="absolute top-0 left-0"/>
<img src="/1-1.webp" v-click class="absolute top-0 left-0"/>
<img src="/1-2.webp" v-click class="absolute top-0 left-0"/>
<img src="/1-3.webp" v-click class="absolute top-0 left-0"/>
<img src="/1-4.webp" v-click class="absolute top-0 left-0"/>
<img src="/2-1.webp" v-click class="absolute top-0 left-0"/>
<img src="/2-2.webp" v-click class="absolute top-0 left-0"/>
<img src="/2-3.webp" v-click class="absolute top-0 left-0"/>
<img src="/3-0.webp" v-click class="absolute top-0 left-0"/>
<img src="/3-1.webp" v-click class="absolute top-0 left-0"/>
<img src="/3-2.webp" v-click class="absolute top-0 left-0"/>
<img src="/3-3.webp" v-click class="absolute top-0 left-0"/>
<img src="/3-4.webp" v-click class="absolute top-0 left-0"/>
</div>

---
layout: two-col
---

## **v2.0.0:** 8×8 blocked and reorder accesses inside a block

I did use *macros*.

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_reorder - 8*8 blocked and reorder accesses inside a block
 *
 * Score: 7.6 + 8.0 + 8.3 = 23.9
 */
char trans_blocked_reorder_desc[] =
    "8*8 blocked and reorder accesses inside a block";
#define a(di, dj) A[i + di][j + dj]
#define b(dj, di) B[j + dj][i + di]
#define line_up(l)                                                             \
  do {                                                                         \
    t0 = a(l, 0);                                                              \
    t1 = a(l, 1);                                                              \
    t2 = a(l, 2);                                                              \
    t3 = a(l, 3);                                                              \
    t4 = a(l, 4);                                                              \
    t5 = a(l, 5);                                                              \
    t6 = a(l, 6);                                                              \
    t7 = a(l, 7);                                                              \
    b(0, l + 0) = t0;                                                          \
    b(1, l + 0) = t1;                                                          \
    b(2, l + 0) = t2;                                                          \
    b(3, l + 0) = t3;                                                          \
    b(0, l + 4) = t4;                                                          \
    b(1, l + 4) = t5;                                                          \
    b(2, l + 4) = t6;                                                          \
    b(3, l + 4) = t7;                                                          \
  } while (0)
#define line_diag(l)                                                           \
  do {                                                                         \
    t0 = a(4, l);                                                              \
    t1 = a(5, l);                                                              \
    t2 = a(6, l);                                                              \
    t3 = a(7, l);                                                              \
    t4 = b(l, 4);                                                              \
    t5 = b(l, 5);                                                              \
    t6 = b(l, 6);                                                              \
    t7 = b(l, 7);                                                              \
    b(l, 4) = t0;                                                              \
    b(l, 5) = t1;                                                              \
    b(l, 6) = t2;                                                              \
    b(l, 7) = t3;                                                              \
    b(l + 4, 0) = t4;                                                          \
    b(l + 4, 1) = t5;                                                          \
    b(l + 4, 2) = t6;                                                          \
    b(l + 4, 3) = t7;                                                          \
  } while (0)
#define line_down(l)                                                           \
  do {                                                                         \
    t0 = a(l + 4, 4);                                                          \
    t1 = a(l + 4, 5);                                                          \
    t2 = a(l + 4, 6);                                                          \
    t3 = a(l + 4, 7);                                                          \
    b(4, l + 4) = t0;                                                          \
    b(5, l + 4) = t1;                                                          \
    b(6, l + 4) = t2;                                                          \
    b(7, l + 4) = t3;                                                          \
  } while (0)
#define line_full(l)                                                           \
  do {                                                                         \
    t0 = a(l, 0);                                                              \
    t1 = a(l, 1);                                                              \
    t2 = a(l, 2);                                                              \
    t3 = a(l, 3);                                                              \
    t4 = a(l, 4);                                                              \
    t5 = a(l, 5);                                                              \
    t6 = a(l, 6);                                                              \
    t7 = a(l, 7);                                                              \
    b(0, l) = t0;                                                              \
    b(1, l) = t1;                                                              \
    b(2, l) = t2;                                                              \
    b(3, l) = t3;                                                              \
    b(4, l) = t4;                                                              \
    b(5, l) = t5;                                                              \
    b(6, l) = t6;                                                              \
    b(7, l) = t7;                                                              \
  } while (0)
void trans_blocked_reorder(int M, int N, int A[N][M], int B[M][N]) {
  int i, j;
  int t0, t1, t2, t3, t4, t5, t6, t7;

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (j = 0; j + 8 <= M; j += 8) {
    for (i = 0; i + 8 <= N; i += 8) {
      line_up(0);
      line_up(1);
      line_up(2);
      line_up(3);
      line_diag(0);
      line_diag(1);
      line_diag(2);
      line_diag(3);
      line_down(0);
      line_down(1);
      line_down(2);
      line_down(3);
    }
    for (; i < N; i++)
      line_full(0);
  }

  for (; j < M; j++) {
    for (i = 0; i < N; i++)
      B[j][i] = A[i][j];
  }

  ENSURES(is_transpose(M, N, A, B));
}
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 | 2244  |  316   | <MissRate :hits="2244" :misses="316"/>  |
| 64×64 | 9064  |  1176  | <MissRate :hits="9064" :misses="1176"/> |
| 60×68 | 8184  |  1768  | <MissRate :hits="8184" :misses="1768"/> |

Score: 7.6 + 8.0 + 8.3 = 23.9

---
layout: two-col
---

## **v2.0.1:** 1×8 blocked, 8×1 loop unroll, and special hack for N=64

εὕρηκα!

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_8_way_ext - 1*8 blocked, 8-way loop unrolling, and special hack
 * for N=64.
 *
 * Score: 8.0 + 8.0 + 10.0 = 26.0
 */
char trans_blocked_8_way_ext_desc[] =
    "1*8 blocked, 8-way loop unrolling, and special hack for N=64";
void trans_blocked_8_way_ext(int M, int N, int A[N][M], int B[M][N]) {
  if (N % 64 != 0)
    trans_blocked_8_way(M, N, A, B);
  else
    trans_blocked_reorder(M, N, A, B);
}
```

::right::

|       | Hits  | Misses |                Miss Rate                |
| :---: | :---: | :----: | :-------------------------------------: |
| 32×32 | 1764  |  284   | <MissRate :hits="1764" :misses="284"/>  |
| 64×64 | 9064  |  1176  | <MissRate :hits="9064" :misses="1176"/> |
| 60×68 | 6658  |  1502  | <MissRate :hits="6658" :misses="1502"/> |

Score: 8.0 + 8.0 + 10.0 = 26.0

---
layout: two-col
---

## **v2.1.0:** blocked and reorder accesses, optimized version

Optimize the diagonal blockes where A and B are conflitcting.

::left::

```c {all} {maxHeight: '380px'}
/*
 * trans_blocked_reorder - 8*8 blocked and reorder accesses inside a block,
 * optimized version
 *
 * Score: 8.0 + 8.0 + 1.8 = 17.8
 */
char trans_blocked_reorder_opt_desc[] =
    "8*8 blocked and reorder accesses inside a block, optimized version";
#define sj (j == 0 ? 8 : 0)
#define da(di, dj) A[j + di][j + dj]
#define db(dj, di) B[j + dj][j + di]
#define sb(dj, di) B[j + dj][sj + di]
#define sline_down(l)                                                          \
  do {                                                                         \
    t0 = da(l + 4, 0);                                                         \
    t1 = da(l + 4, 1);                                                         \
    t2 = da(l + 4, 2);                                                         \
    t3 = da(l + 4, 3);                                                         \
    t4 = da(l + 4, 4);                                                         \
    t5 = da(l + 4, 5);                                                         \
    t6 = da(l + 4, 6);                                                         \
    t7 = da(l + 4, 7);                                                         \
    sb(l, 0) = t0;                                                             \
    sb(l, 1) = t1;                                                             \
    sb(l, 2) = t2;                                                             \
    sb(l, 3) = t3;                                                             \
    sb(l, 4) = t4;                                                             \
    sb(l, 5) = t5;                                                             \
    sb(l, 6) = t6;                                                             \
    sb(l, 7) = t7;                                                             \
  } while (0)
#define sline_up(l)                                                            \
  do {                                                                         \
    t0 = da(l, 0);                                                             \
    t1 = da(l, 1);                                                             \
    t2 = da(l, 2);                                                             \
    t3 = da(l, 3);                                                             \
    t4 = da(l, 4);                                                             \
    t5 = da(l, 5);                                                             \
    t6 = da(l, 6);                                                             \
    t7 = da(l, 7);                                                             \
    db(l, 0) = t0;                                                             \
    db(l, 1) = t1;                                                             \
    db(l, 2) = t2;                                                             \
    db(l, 3) = t3;                                                             \
    db(l, 4) = t4;                                                             \
    db(l, 5) = t5;                                                             \
    db(l, 6) = t6;                                                             \
    db(l, 7) = t7;                                                             \
  } while (0)
#define strans(bname, l)                                                       \
  do {                                                                         \
    t0 = bname(0, l + 1);                                                      \
    t1 = bname(0, l + 2);                                                      \
    t2 = bname(0, l + 3);                                                      \
    t3 = bname(1, l + 2);                                                      \
    t4 = bname(1, l + 3);                                                      \
    t5 = bname(2, l + 3);                                                      \
    bname(0, l + 1) = bname(1, l + 0);                                         \
    bname(0, l + 2) = bname(2, l + 0);                                         \
    bname(0, l + 3) = bname(3, l + 0);                                         \
    bname(1, l + 2) = bname(2, l + 1);                                         \
    bname(1, l + 3) = bname(3, l + 1);                                         \
    bname(2, l + 3) = bname(3, l + 2);                                         \
    bname(1, l + 0) = t0;                                                      \
    bname(2, l + 0) = t1;                                                      \
    bname(3, l + 0) = t2;                                                      \
    bname(2, l + 1) = t3;                                                      \
    bname(3, l + 1) = t4;                                                      \
    bname(3, l + 2) = t5;                                                      \
  } while (0)
#define sdswap(l)                                                              \
  do {                                                                         \
    t0 = db(l, 4);                                                             \
    t1 = db(l, 5);                                                             \
    t2 = db(l, 6);                                                             \
    t3 = db(l, 7);                                                             \
    db(l, 4) = sb(l, 0);                                                       \
    db(l, 5) = sb(l, 1);                                                       \
    db(l, 6) = sb(l, 2);                                                       \
    db(l, 7) = sb(l, 3);                                                       \
    sb(l, 0) = t0;                                                             \
    sb(l, 1) = t1;                                                             \
    sb(l, 2) = t2;                                                             \
    sb(l, 3) = t3;                                                             \
  } while (0)
#define sline_back(l)                                                          \
  do {                                                                         \
    t0 = sb(l, 0);                                                             \
    t1 = sb(l, 1);                                                             \
    t2 = sb(l, 2);                                                             \
    t3 = sb(l, 3);                                                             \
    t4 = sb(l, 4);                                                             \
    t5 = sb(l, 5);                                                             \
    t6 = sb(l, 6);                                                             \
    t7 = sb(l, 7);                                                             \
    db(l + 4, 0) = t0;                                                         \
    db(l + 4, 1) = t1;                                                         \
    db(l + 4, 2) = t2;                                                         \
    db(l + 4, 3) = t3;                                                         \
    db(l + 4, 4) = t4;                                                         \
    db(l + 4, 5) = t5;                                                         \
    db(l + 4, 6) = t6;                                                         \
    db(l + 4, 7) = t7;                                                         \
  } while (0)
void trans_blocked_reorder_opt(int M, int N, int A[N][M], int B[M][N]) {
  int i, j;
  int t0, t1, t2, t3, t4, t5, t6, t7;

  REQUIRES(M > 0);
  REQUIRES(N > 0);

  for (j = 0; j + 8 <= M; j += 8) {
    if (j + 8 <= N && 16 <= N) {
      sline_down(0);
      sline_down(1);
      sline_down(2);
      sline_down(3);
      sline_up(0);
      sline_up(1);
      sline_up(2);
      sline_up(3);
      strans(db, 0);
      strans(db, 4);
      strans(sb, 0);
      strans(sb, 4);
      sdswap(0);
      sdswap(1);
      sdswap(2);
      sdswap(3);
      sline_back(0);
      sline_back(1);
      sline_back(2);
      sline_back(3);
    }
    for (i = 0; i + 8 <= M; i += 8)
      if (j != i || 16 > N) {
        line_up(0);
        line_up(1);
        line_up(2);
        line_up(3);
        line_diag(0);
        line_diag(1);
        line_diag(2);
        line_diag(3);
        line_down(0);
        line_down(1);
        line_down(2);
        line_down(3);
      }
    for (; i < N; i++)
      line_full(0);
  }

  for (; j < N; j++) {
    for (i = 0; i < N; i++)
      B[j][i] = A[i][j];
  }

  ENSURES(is_transpose(M, N, A, B));
}
```

::right::

|       | Hits  | Misses |                Miss Rate                 |
| :---: | :---: | :----: | :--------------------------------------: |
| 32×32 | 3072  |  256   |  <MissRate :hits="3072" :misses="256"/>  |
| 64×64 | 10752 |  1024  | <MissRate :hits="10752" :misses="1024"/> |
| 60×68 | 9739  |  2421  | <MissRate :hits="9739" :misses="2421"/>  |

Score: 8.0 + 8.0 + 1.8 = 17.8

---
layout: two-col
---

## **v2.1.1:** Final version

The final rank is 11.

::left::

```c {all} {maxHeight: '380px'}
char transpose_submit_desc[] = "Transpose submission";
void transpose_submit(int M, int N, int A[N][M], int B[M][N]) {
  if (N % 8 != 0)
    trans_blocked_8_way(M, N, A, B);
  else
    trans_blocked_reorder_opt(M, N, A, B);
}
```

::right::

|       | Hits  | Misses |                Miss Rate                 |
| :---: | :---: | :----: | :--------------------------------------: |
| 32×32 | 3072  |  256   |  <MissRate :hits="3072" :misses="256"/>  |
| 64×64 | 10752 |  1024  | <MissRate :hits="10752" :misses="1024"/> |
| 60×68 | 6658  |  1502  | <MissRate :hits="6658" :misses="1502"/>  |

Score: 8.0 + 8.0 + 10.0 = 26.0
