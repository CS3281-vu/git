# Parallelizing Git

## Overview 

With the increasing prevalence of multicore computers, it is only natural that there is potential for embarrassingly parallel operations to massively increase performance for users, by cutting down the time for an operation by a factor of 2 or 4. Git, a very common DVCS tool, has a lot of use cases where it benefits from parallelization: delta compression, uploading and downloading packfiles, and so on.

There are some Git operations that currently do not make use of parallelization, however, and the goal of this project will be to parallelize those operations and determine if there is any performance benefit.

##  Goals and Scope

There are two operations which, after speaking with a core Git maintainer (Jeff King, a.k.a. Peff), we have decided to attempt parallelizing:

diffs involving multiple files
determining whether files have been renamed


For the sake of a final project for operating systems, it is important to note that in the parallelized code in pack-objects.c takes advantage of pthreads, mutex locking structures, and condition variables to perform concurrent execution. Although we have not yet designed our implementation, we imagine that we will be taking advantage of all three of these core concepts from the operating systems curriculum. Moreover, we believe that using this project as an opportunity to contribute to an open source, powerful, widely used VCS is a valuable learning experience for both of us as we familiarize ourselves with the existing codebase, design a solution, and implement and attempt to merge our solution.
We expect to modify [diff.c](https://github.com/git/git/blob/v2.12.2/diff.c#L4734-L4857) and [diffcore-rename.c](https://github.com/git/git/blob/v2.12.2/diffcore-rename.c#L130-L206).

## Architecture

Per Peff’s recommendations, Git uses the following threading model for parallelization, which we will seek to implement:

`Our usual approach is to have a worker-pool model. In the threaded case,
we have N threads and we feed each of them a bit of work, and then wait
for results to come in. In the non-threaded case, we just do the work
ourselves. See the way that pack-objects handles delta compression.`

[Link to relevant code in builtin/pack-objects.c.](https://github.com/git/git/blob/v2.12.2/builtin/pack-objects.c#L2111-L2238)

Notably, Git is also designed to be backwards-compatible with very old compilers - pre-C89, in fact - and as such it must also be the case that the code must be written in such a fashion to [support compiling Git with pthread support disabled](https://github.com/git/git/blob/v2.12.2/Makefile#L181).

## Testing

- [https://github.com/git/git/blob/v2.12.2/t/t4000-diff-format.sh](https://github.com/git/git/blob/v2.12.2/t/t4000-diff-format.sh)
- [https://github.com/git/git/blob/v2.12.2/t/t4001-diff-rename.sh](https://github.com/git/git/blob/v2.12.2/t/t4001-diff-rename.sh)
- [https://github.com/git/git/blob/v2.12.2/t/t4002-diff-basic.sh](https://github.com/git/git/blob/v2.12.2/t/t4002-diff-basic.sh)
- … 
- [https://github.com/git/git/blob/v2.12.2/t/t4062-diff-pickaxe.sh](https://github.com/git/git/blob/v2.12.2/t/t4062-diff-pickaxe.sh)

If we are able to achieve performance gains, we would love to merge upstream and have our code be part of the core Git codebase. Peff has indicated that even if performance gains are not achieved or are not notable, the information gathered as we test will be valuable for the team as it considers future directions on parallelizability. For how performance will be measured, see the section below.

## Performance

- [https://github.com/git/git/blob/v2.12.2/t/perf/p4000-diff-algorithms.sh](https://github.com/git/git/blob/v2.12.2/t/perf/p4000-diff-algorithms.sh)
- Benchmark options: kernel diffs, git source diffs - any other real-world use cases?
- How do the various GUIs do big diffs? Might be of interest to ask that.
      - How to benchmark perf changes in estimated_similarity()?

Before and after comparisons will be used to shown the performance of the parallelized tasks.


#### Side Note

A couple of considerations have been taken into account for the purposes of our design. Notably, proper memory management strategies are needed to prevent client machines from being taken over when running enormous diffs.

- Set resource limits to prevent the parallel diffing from taking up all memory on an entire machine
- Using control Groups as a pattern for limiting memory

