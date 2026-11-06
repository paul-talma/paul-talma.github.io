---
layout: post
title: intro to heartbeat scheduling
---

# Heartbeat Scheduling

Heartbeat schedling is a simple yet efficient technique for controlling thread-creation overhead in parallel code.

The basic idea is to amortize the overhead by restricting thread creation to periodic "heartbeats."
Since the code runs serially between heartbeats, the cost of thread creation is offset by serial work.
Heartbeat scheduling is easy to understand and implement, yet achieves provably bounded scheduling overheads.

## The scheduler

Heartbeat scheduling uses a thread scheduler.
In this framework, a **thread** (sometimes called "lightweight thread," "job," or "task") is a unit of work.
The scheduler assigns threads to **workers** (constituting a **thread pool**), which execute threads on a **processor**.
Workers are typically mapped one-to-one onto OS threads.
In the lifetime of the program, a single OS thread will run a single worker, which will in turn run many threads (no, I don't like this nomenclature either).

To mitigate load imbalances, we will use a [work-stealing](https://en.wikipedia.org/wiki/Work_stealing) scheduler.
Each worker has a private [deque](https://en.wikipedia.org/wiki/Double-ended_queue) of threads from which it can `pop()` and `run()` a thread.
If its own deque is empty, it will try to `steal()` a thread from another worker.
This prevents worker starvation.

Schematically:

| worker1 | worker2 | worker3 |
| ------- | ------- | ------- |
| thread1 |         | thread3 |
| thread2 |         | thread4 |
|         |         |         |

Here, worker 2 has finished its work and so will attempt to steal from another worker with pending threads.

## Fork-Join Parallelism

Heartbeat is best understood in the context of **fork-join parallelism**.
Fork-join parallelism is particularly well-suited for parallelizing divide-and-conquer algorithms.
Divide-and-conquer algorithms have the following schematic form:

```
def solve(problem):
    if problem is small enough:
        return answer(problem)

    p1, p2 = partition(problem)
    return combine(solve(p1), solve(p2))
```

For example, consider the problem of summing a binary tree:

```c
struct node {
    double val;
    node *left;
    node *right;
};
```

A divide-and-conquer algorithm might go like this:

```c
double sum(struct node *n) {
    if (!n) {
        return 0;
    }
    return sum(n->left) + sum(n->right) + n->val;
}
```

In a fork-join implementation, at least one of the recursive calls becomes a thread handed to the scheduler.
Schematically,

```
def solve_fj(problem):
    if problem is small enough:
        return answer(problem)

    lower, upper = partition(problem)
    t = make_thread(upper)
    fork(t)
    lower_ans = solve_fj(lower)
    join(t)
    return combine(lower_ans, t.result)
```

Here, we first use `make_thread` to package the upper half of our problem in to a thread.
The `fork(t)` call passes the thread to the scheduler, which is in charge of finding a worker to run it on.
The current thread then solves the lower half of our problem and `join`s with the upper half.
`join(t)` signals to the scheduler that the current worker is waiting for `t` to complete and can perform useful work while it waits.
Once `t` finishes, we get the solution `t.result` to the upper half of the problem and `combine` it with the solution to the lower half.

Continuing our example, a fork-join implementation of the binary tree sum might look like

```c
double sum_fj(node *n) {
    if (!n)
        return 0;

    thread t = make_thread(n->right);
    fork(&t);
    double left_sum = sum_fj(n->left);
    join(&t);
    return left_sum + t.result + n->val;
}
```

As you can see, divide-and-conquer alorithms lend themselves well to fork-join parallelization.
Note that this implementation is somehwhat naive.
For example, we create a thread for the right branch even when `node->right == NULL`, incurring thread creation and scheduling overhead, only for the worker executing that thread to immediately return `0`.
We could instead check whether the right child is `NULL` and skip the fork if it is.
A more serious inefficiency concerns the **task granularity**.
Our program creates a thread for each node in the tree, and each thread does very little work: return `0` if at a leaf node, or recurse and sum, resulting in low thread utilization.
These overheads mean that on many inputs, our fork-join algorithm performs worse than our serial one!

We can remedy this problem in several ways.
The simplest is to limit the number of threads we create.
For example, we can limit thread creation to the first `N` levels of the recursion.

Here is an improved fork-join algorithm, incorporating these fixes:

```c
double sum_fj_improved(node *n, size_t depth, size_t max_depth) {
    if (!n)
        return 0;

    if (depth == max_depth) { // granularity control
        return sum(n);
    }
    if (!n->right) { // don't fork if right child is NULL
        return sum_fj_improved(n->left, depth + 1, max_depth) + n->val;
    }
    thread t = make_thread(n->right, depth + 1, max_depth);
    fork(&t);
    double left_sum = sum_fj_improved(n->left, depth + 1, max_depth);
    join(&t);
    return left_sum + t.result + n->val;
}
```

As the following graph shows, the choice of `N` is very important.
Too large, and we drown in overheads.
Too small, and we fail to exploit parallelism as workers deal with large, possibly unbalanced workloads.
Obviously, the optimal value of `N` depends on the input data and cannot be known ahead of time.

-- TODO: insert graph

To compare what we have so far, let's benchmark against a few sample trees:

-- TODO: insert benchmark table

As expected, our optimized fork-join implementation performs better than our naive one.
However, in several cases, both fall short of the serial implementation.

## Hearteat Scheduling
