+++
title = "Understanding Vyukov MPMC ring buffer"
description = "Markdown Syntax test page"
date = "2025-01-05"
tags = ["concurrency"]
+++

## Intro

Recently I had to create a simple MPMC ring buffer for inter-cpu communication based on shared memory between them.
For simplicity let's imagine N threads, which share same buffer and exchange messages of the fixed size. Quick search in known
concurrency-related blogs showed me very simple ring buffer by [Dmitry Vyukov](https://www.1024cores.net/home/lock-free-algorithms/queues/bounded-mpmc-queue).

In this blog post I will summarize how it works.

## Bounded MPMC Queue

MPMC stands for multi-producer and multi-consumer queue, which means that queue support multiply concurrent readers and writers while reversing
consistency. Bounded means that queue can hold only fixed number of messages. If queue is full, then push will fail.

Dmitry's design uses fixed array of structures called cells:

(I am using C-ish pseudo-code, not a real language)

```c
struct Cell<T> {
    atomic<size_t> sequence_number;
    T data;
};
```

Queue itself contains classical head, tail and number of elements. For performance reasons size of the array should be power of 2. This causes module operation like
`idx % size` to compile into single `and` instruction.

```c
struct Queue<T> {
    Cell *data;
    atomic<size_t> head;
    atomic<size_t> tail;
    size_t size_mask;
};
```

### Initialization

During construction each cell is initialized with `sequence_number` equal to it's position index.

```c
    for (size_t i = 0; i < num_elements; ++i)
        atomic_store_explicit(&queue->data[i].seq, i, __ATOMIC_RELAXED);
```

`head` and `tail` indexes are initialized to 0.

In memory this looks like (for simplicity let's take array size equal to 2)

```
+---+   +---+
| 0 | - | 1 |
+---+   +---+

head = 0;
tail = 0;
```

### Pushing

Push is only possible if `data[head].sequence_number` is equal to `head`. On successful push, `head` and `sequence_number` indexes are incremented.
However if they are not equal there are 2 possible cases:

1) `head` > `sequence_number`. In such case queue is simply full. Let's take an example from image above. If we push 2 elements queue will have following state:

```
+---+   +---+
| 1 | - | 2 |
+---+   +---+

head = 2;
tail = 0;
```

2) If `head` < `sequence_number`. Such case means push contention and we need to retry. This could happen in following case:

```
                CPU0                                CPU1

x0 = read_head()                               x0 = read_head()
x1 = read_seq_number()
if (x0 == x1)   // EQUALS
   cas(seq, seq, seq + 1)
                                               x1 = read_seq_number()
                                               // CPU0 has just updated sequence_number, while x0 hold
                                               // old head index
                                               if (x0 < x1)
                                                  <retry>
```

### Popping

Pop is only possible if `data[tail].sequence_number` is equal to `tail + 1`. This +1 comes from push, since successful push increments
sequence number. On successful pop, `tail` is incremented, while `sequence_number` is assigned to value `tail + array_size`. This step sets
sequence_number equal to suitalbe `head` for pushing. In general it's the next number `N` that satisfies `i = N MOD array_size`, where `i` is
an array index.

However if they are not equal there are also 2 possible cases:

1) `tail + 1` > `sequence_number`. Such case means that queue is empty.

Let's again see some pictures

Empty queue:
```
+---+   +---+
| 0 | - | 1 |
+---+   +---+

head = 0;
tail = 0;
```

Pop is not possible, since `data[tail].sequence_number + 1 > pos`. 

Queue with 1 element:
```
+---+   +---+
| 1 | - | 1 |
+---+   +---+

head = 1;
tail = 0;
```

Now we can pop, since `sequence_number` is equal to 1.

2) `tail + 1` < `sequence_number`. This case indicates contention on pop alike in push case

```
                CPU0                                CPU1

x0 = read_tail()                               x0 = read_tail()
x1 = read_seq_number()
if (x0 + 1 == x1)   // EQUALS
   cas(seq, seq, seq + array_size)
                                               x1 = read_seq_number()
                                               // CPU0 has just updated sequence_number, while x0 hold
                                               // old tail index
                                               if (x0 + 1 < x1)
                                                  <retry>
```
