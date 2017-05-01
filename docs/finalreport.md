# Final Report: Parallelizing Git’s File Rename Recognition

An experiment in parallelizing git rename resolution by Samuel Lijin and Harrison Stall

## Original Goals

Since the purpose of this project was to demonstrate proficiency with and understanding of concepts learned in this course - particularly parallelism concepts - we decided to look into seeing if there were any git operations that could potentially benefit from parallelism. Most of the low-hanging fruit - packfile generation, delta compression, grep, and so on - has already been parallelized, so we reached out to Jeff King (a.k.a. Peff), a core git maintainer for suggestions about what we might be able to work on.

Peff suggested two places where there was potential for parallelism to improve git performance: diffs spanning multiple files and rename determination. From an abstract perspective, both operations seem to be relatively parallelizable: they consist of numerous individual computations which do not depend upon one another so it seems it should be more than possible to implement some modicum of parallelization in both cases.

## Investigating the Parallelizability of Multi-file Diffs

However, “should” does not necessarily mean “is”, particularly when a codebase has complex, decade-old dependencies. In the case of the former - parallelizing multi-file diffs - after digging into the codebase, we realized that this was a significantly more complicated task than we imagined it to be.

Git currently (as of v2.12.2) diffs multiple files sequentially by looping through the files to be diffed, and during each loop, computes that diff and flushes it to stdout. To parallelize this operation, then, would require adjusting this operation to first compute every diff, synchronize on the completion of all diff computations, and then flush all diffs to stdout.

This is uniquely challenging for two reasons in particular. The first is that git has implemented support for the computation of various diff formats (patch, raw, stat, etc) through the use of multiple code paths invoked from the main loop involved in diff generation (see `diff.c:diff_flush()`). As such, significant architectural refactoring would have been necessary to modify these codepaths to support asynchronous computation.

It is possible that we could have surmounted the first challenge involved; however, the second rendered this impossible: the entire diff codepath is coded around the assumption that computation and output generation are tightly coupled. In fact, git also supports the use of external diff drivers (see `diff.c:run_diff()`, `diff.c:run_diff_cmd()`, `diff.c:run_external_diff()`, `run-command.c:run_command_v_opt_cd_env()`) but does not support capturing the stdout of a forked command in its run-command API (see `Documentation/technical/api-run-command.txt`), which means supporting parallelism with external diff drivers would require extensive modification in that part of the codebase.

Even if we had been willing to forego supporting parallelism with external diff drivers, implementing parallelism with the builtin diff driver would have been a similarly imposing challenge. Git wisely did not roll their own library for computing diffs, but instead incorporates the xdiff library to do so; however, the manner in which git invokes xdiff routines to compute diffs, again, is coded around the assumption that computation and output generation are tightly coupled.

After spending far too many hours digging around the guts of xdiff, we realized that parallelizing multi-file diffs - especially if we wanted to merge our work upstream - was nigh impossible, and is without a doubt far beyond the scope of an undergraduate final project, even with a month to do it. Doing this properly would have involved extensive architectural changes (not just within the diff codepaths, but also in the run-command API; it also had the potential to involve breaking changes in requirements for external diff drivers). Instead, we opted to take the second route: exploring the parallelization of git’s rename computation algorithm.

## Parallelizing Rename Computation

### Background

To explain the “rename computation” process, some background about git is first required. Every git commit corresponds to a snapshot of every tracked file in the repository at the time of the commit; commits do not, however, contain any information about file history. For that matter, git _never_ records information about changes: it only records states.

All git changelog information is computed in real-time: if a file was deleted, git determines that in real-time; if a file was added, git determines that in real-time; if a file was renamed, git determines that in real-time. None of this information is recorded in the git database; this was a very [conscious design decision that Linus Torvalds made in the very early days of git](http://public-inbox.org/git/Pine.LNX.4.58.0504150753440.7211@ppc970.osdl.org/).

As such, diffs, logs, and all other git operations that compare files between commits have to compare their git trees (that is, their filetree snapshots) to determine what files were deleted, added, or renamed. This is not as simple as it seems: when comparing two commits, if a file is present in the former commit but not the latter, that does not necessarily mean the file was deleted; it can also mean that the file was renamed. Similarly, if a file is present in the latter commit but not the former, that does not necessarily mean the file was added - it can also mean that the file is the result of a rename.

This is additionally complicated by the fact that between two commits, a file may not only have had its name changed, but its contents may also have been changed. Thus, when computing changes between two commits, git internally estimates similarities between files in the two commits (`diffcore-rename.c:estimate_similarity()`) and uses the estimated scores to determine if a pair of files between the two commits should or should not be treated as a file rename.

Specifically, inside `diff.c` (which contains the source code invoked by any git diff operation) the `diffcore_std()` function used for building diffs throughout the codebase calls `diffcore-rename.c:diffcore_rename()`, which determines if file renames happened between the two specified commits. By constructing an m by n matrix of all possible renames (from candidate source files and candidate destination files, in the former and latter commit respectively), filling in the matrix with similarity scores (`struct diff_score`) which proxy for the likelihood of a given source and destination pair corresponding to a rename, and then sorting this matrix, one can determine which files were likely renamed between commits.

It is this computation that we worked on parallelizing, since multiple instances of it are run per diff, many of which can be run simultaneously (every potential rename source has to be compared against every potential rename destination, so some degree of synchronization is necessary). We assumed that if successful, this would prove particularly useful on modern machines when computing diff_scores for large matrices, since the standard consumer computer generally has at least two cores, if not more. By partitioning diff_score computation amongst threads, it should be possible to achieve noticeable speedups. Peff suggested modelling our parallelization model off the thread pool model used for delta compression (see `builtin/pack-objects.c:ll_find_deltas()`); given the complexity of this threading model, however, we elected to first attempt alternative approaches.

### Attempt 1: Lock-Free and Bounded Concurrency

We first attempted a lock-free approach with bounded threading (instead of a thread pool, simply cap the number of concurrent threads). Some of this work can be found on the [`broken.parallelize`](https://github.com/CS3281-vu/git/commits/broken.parallelize) branch. The idea behind this approach was to first load file blobs from the git database prior to computing similarities, since git currently loads blobs lazily during similarity computation; however, this turned out to be error prone for a number of reasons and did not pass the git test suite. These reasons for failure are not entirely clear to us; it is possible that the lazy-loading is not as idempotent as we assumed it was. In addition, the lazy loading is currently designed to not load file blobs to compute similarities if the sizes differ by more than a certain threshold.

### Attempt 2: Locks and Bounded Concurrency

We then attempted a bounded threading approach with locking; this work can be found on the [`parallelize.windows`](https://github.com/CS3281-vu/git/commits/parallelize.windows) branch (note that this branch was rebased onto a version of git used to build Git for Windows and does not necessarily build on Linux). In this approach, we instead simply modify the similarity computation to lock the rename source and rename destination that it is comparing (this is necessary because we preserve the lazy-loading behavior of `diffcore-rename.c:estimate_similarity()`). Deadlock was avoided by applying a total order to the locking: we consistently lock the potential source file first, then the potential destination second.

Although this version passes the test suite, its performance is significantly worse than the current version of git, for two reasons: first, it increases the computational complexity because a new thread is set up and taken down and locks have to be acquired and released for every computation. Second, the order of thread execution (for every potential destination, for every potential source, compute similarity) guarantees that almost every single thread will be competing with another thread for a lock at any given point in time, since computations comparing many sources to one destination are kicked off in that order.

### Attempt 3: Locks, Bounded Concurrency, and Modified Computation Order

To mitigate this, we looked into traversing the rename matrix along its diagonals; that is, if you envision the table of source/destination comparisons that have to be computed every time as a 2D array of scores to be filled in, traversing this table column by column or row by row is naturally lock-inefficient because computations involving the same source or destination cannot happen concurrently. In code it looks something like this:

~~~
void compute_diff_score(struct diff_score *diff_score_matrix, column, row);
struct diff_score *matrix = xcalloc(columns * rows, sizeof(diff_score));
int i, j;
for (i = 0; i < columns; i++) {
    for (j = 0; j < rows; j++) {
        // instead of iterating through the matrix column by column,
        // iterate through it “diagonal” by “diagonal”
        compute_diff_score(matrix, (i + j) % columns, j);
    }
}
~~~

This work can be found on [`parallelize.reorder`](https://github.com/CS3281-vu/git/commits/parallelize.reorder). It turns out, unfortunately, that although this improved concurrency because threads were not constantly competing with one another for locks, the sheer number of computations involved means that spawning a new thread per computation is wholly unsustainable in terms of performance - git operations that used to take seconds now took multiple minutes.

### Attempt 4: Locks and Thread Pool - Success!

The next step, naturally, was to revisit the concurrency design. Since now the blocking source of overhead had become thread setup and teardown - and recalling Peff’s original recommendation - we elected to switch from the thread bounding approach we had taken to a proper thread pool model. The first attempt was a disastrous failure: we attempted to set up the thread pool as a pool of consumers, each with their own work queue, modelled off similar designs elsewhere in the git codebase (see `builtin/pack-objects.c:ll_find_deltas()`).

We quickly realized that this was an unnecessarily complex approach with many deceptive corner cases and that there was a much simpler solution: instead of assigning each consumer a queue, we could design the consumers to treat the diff_score matrix as a concurrent queue. This is the first approach that we managed to succeed with (see [`parallelize.thread-pool.0`](https://github.com/CS3281-vu/git/commits/parallelize.thread-pool.0)), and in fact is the backbone behind the approach we ultimately settled on.

Due to the complexity of the git source code, however, and because many of the dependencies were not designed for concurrent access, we had to make three additional modifications. First, some metadata which had previously been lazy-loaded was now actively loaded, prior to starting the worker threads; second, modifications of the attribute stack (the data structure used to keep track of information in .gitconfig and .gitattributes files) were synchronized using a mutex (see `attr.c:prepare_attr_stack()`); and third, packfile decompression (decompression of compressed information in the git database) was synchronized with a reentrant mutex (see `sha1_file.c:read_packed_sha1()`).

### Attempts at Improvement

The first working thread pool implementation, however, when benchmarked against diffs of consecutive Linux kernel versions, spent significantly more time in userspace than the current version of git does. (The system time also increased, of course, but this is only to be expected because of threading.) We suspected a number of reasons for this increased time: cache misses, page faults (which went from ~500,000 to ~1,000,000 between v2.12.2 and our thread pool version for the duration of the benchmark), and locking. Two modifications were tested, in various combinations, to see if they would address these issues.

The first modification we tested was adjusting the queue access pattern. The thread pool originally preserved the diagonal-order access pattern described in the third attempt; however, an astute observer will note that although this may have resolved the original concurrency issue it was implemented to address, this is also a fantastic way to thrash the cache, especially when the matrix may have millions of entries (in cases where there are more than a thousand source candidates and more than a thousand destination candidates for every rename).

To address this, we decided to have the worker threads consume not individual jobs to compute from the diff_score matrix, but rather sets of jobs: specifically, threads would consume all computations comparing every source to a given destination during a single consumption step. This has the added benefit of enabling us to reduce the locking overhead, since the destination diff_filespec only needs to be locked once. More importantly, though, since a thread does all computations involving a single destination diff_filespec in one go, it can keep that diff_filespec in its cache, which greatly improves caching performance. This ended up significantly reducing the time spent in userspace (although only about halfway to the time spent in userspace by the current version of git) (see `parallelize.thread-pool.1`).

The second modification that we tested was lazy locking: since the primary reason we had to lock every diff_filespec was because information is lazy loaded during diff_score computation, we assumed that if we only locked a diff_filespec if its information had not been loaded, we might have seen performance gains. Unfortunately, this turned out not to be the case: in fact, performance actually slightly _degraded_ when we implemented lazy locking. Although we are not certain why, our best guess is that (1) mutex acquisition is not a particularly expensive operation (on the order of 25ns), and (2) by acquiring locks lazily, we wreak havoc on branch prediction, which ends up being more expensive. (This remained the case with both access patterns, see `parallelize.thread-pool.2` and `parallelize.thread-pool.3`.)

### Notes

Unfortunately, neither of our machines have more than two cores so we could not test the scenarios in which this could be the most beneficial. Thus, with two concurrent rename-computing threads, the performance was still slightly worse than the current single-threaded model used by git (see Testing section for the full set of sample data).

The bulk of the changes we made were inside `diffcore-rename.c`, particularly inside `diffcore_rename()`. We also added locks and minor modifications to `attr.c`, and `sha1-file.c`. Inside `diffcore_rename.c`, we added a new function to pass to threads called `threaded_calc_diff_score()`. We pass in a struct of thread parameters called `calc_diff_score_thread_params`. The maximal number of active threads is set at a ceiling of `online_cpus()`, a function from `thread-utils.c` in the git codebase which returns the number of cores.

## Testing

The nice thing about working with an existing large-scale open-source codebase is that full-coverage test suites are provided for ensuring that changes are not breaking when pull requests are made. Thus, as we made changes we could continuously test our changes to the codebase, ensuring that our changes were not breaking. The downside is that the test suite contains thousands of tests and takes several minutes at best when executing the entire suite.

### Git Test Suite

To kick off the tests, clone the repository and simply run `$ make && make test`. Interestingly, the maintainers have several tests marked broken that fail even on their current production-pushed branches; they are marked with `# TODO known breakage` comments and after passing the required tests append a comment saying `# still have <n> known breakage(s)` where n is the number of broken tests. When tests that are known not to be broken are failed, test execution is stopped. 

With the quantity of tests and scope of the testing, debugging test cases got tricky. After grappling with environment issues when trying to recreate breakage state, we found that certain errors were not easily reproducible. In response, Sam decided to reach out to Peff asking how they reproduce test-fail state. This is what we received in response:

> I'd usually start by stopping the script at the failed test, going into the repository directory, and then running from there. Like:
>
> $ cd   
> $ ./t5000-tar-tree.sh -v -i -x
> $ cd trash\ directory.t5000-tar-tree  
> $ gdb --args ../../git whatever-failed
> 
> That's usually enough to re-create the broken state. Sometimes there are things in the environment, or the test is impacted by your user-level config (test-lib resets `$HOME`, so it should have no user config). But it's usually easier to just stick a "gdb" temporarily into the test. Note there are some tricks there with redirections and the bin-wrappers script; see the debug() helper in `test-lib-functions.sh`.

Debugging such an enormous codebase with gdb was definitely not the easiest task but that only made test passes all the more exciting. Plus, this helped us quickly confirm the correctness of our changes.

### Performance Testing

That said, the all encompassing nature of the test suite was a blessing and a curse. Although the tests were useful for assessing correctness, we still needed a separate set of tests to evaluate the impact of our changes under heavy load with respect to correctness and performance. Naturally, we turned to a codebase that is constantly changing and provided a case in which examining diffs could be useful: the Linux kernel. We compared the time taken to perform kernel difs using the most recently released version of git, v2.12.2 and the version built from our own patches, which were applied on top of v2.12.2. The following two scripts were used to assess performance

`single-perf-test-helper.sh`, and

~~~
do_tag_diff () {
    echo Diffing $prev_tag $tag
    if test -z "$NON_CUMULATIVE"
    then
        $GIT_BINARY diff --raw -M -l0 $prev_tag $tag 2>&1 >/dev/null
    else
        /usr/bin/time -v $GIT_BINARY diff --raw -M -l0 $prev_tag $tag 2>&1 >/dev/null
        echo
    fi
}

if test -n "$RUN_ALL"
then
    prev_tag=
    $GIT_BINARY tag | grep -v - -- | sort -V | while read tag
    do
        if test -n "$prev_tag"
        then
            do_tag_diff | tee -a $LOG_FILE
        fi
        prev_tag=$tag
    done
else
    while read prev_tag tag
    do
        do_tag_diff | tee -a $LOG_FILE
    done <<EOF
v2.6.23.17  v2.6.24
v2.6.26.8   v2.6.27
v2.6.27.62  v2.6.28
v2.6.28.10  v2.6.29
v2.6.29.6   v2.6.30
v2.6.31.14  v2.6.32
v2.6.36.4   v2.6.37
v3.0.101    v3.1
v3.1.10     v3.2
v3.6.11     v3.7
v3.14.79    v3.15
v3.16.43    v3.17
v4.4.63     v4.5
EOF
fi
~~~

`single-perf-test.sh`.

~~~
#!/bin/sh

# default: time a cumulative run of only the expensive kernel diffs

# set envvar RUN_ALL to only run all consecutive tag diffs, not just the expensive ones
# set envvar NON_CUMULATIVE to time all diffs separately

# $1 indicates the git binary to use
# $2 indicates the log file to write to

MY_PATH=$(dirname $(realpath -s $0))

export GIT_BINARY=$1
export LOG_FILE=$2

if test -z "$NON_CUMULATIVE"
then
    /usr/bin/time -v ${MY_PATH}/single-perf-test-helper.sh 2>&1 | tee -a $LOG_FILE
else
    ${MY_PATH}/single-perf-test-helper.sh
fi
~~~

The idea is that diffing between consecutive versions of the Linux kernel is a good example of an intensive real-world use for code like this, and that we could get useful data by running only the particularly expensive version diffs (this is the default behavior; a diff was determined to be “expensive” if git v2.12.2 spent over a second in userspace computing it). Correctness was assessed by comparing the output of such version diffs generated by git v2.12.2 and our version; it should be noted that “comparing” does not mean the output has to match precisely, but rather that the output has to be reasonably comparable. It so happens, in fact, that there are some filepairs which git v2.12.2 did not determine to be renames even though they rightly were, and our patched version noticed those.

Our benchmark numbers are also dumped inside the repository. The files named patch-perf#.txt refer to our experimental git version’s performance whereas stable-perf#.txt files refer to git 2.12.2’s performance. Generally, it’s a good idea to only look at the files corresponding to numbers > 2, as that’s generally the threshold at which the cache is warmed up and we are able to get the most accurate depiction of performance.

By comparing the results in the files and comparing performance numbers, we found that git 2.12.2 performed slightly better than our experimental version. The most likely culprit was the page faults incurred in the patched version. In our third experimental run, the difference in time elapsed was only 1.85 seconds with version 2.12.2 running in 1 minute and 43.96 seconds compared to our patched version taking 1 minute and 46.04 seconds. We imagine that storing more filepairs in memory and requiring mutex locks for each filespec could have contributed to this performance loss. Like we mentioned earlier, concurrency can come with added costs even in cases where gains seem to be clearly achievable with slight design alterations. Still, even if not within the extremely small standard deviation of our samples, we are painfully close to achieving performance gains with the multithreaded approach and imagine that running tests on a 4-core machine could significantly improve the results of our patches. The data from three trials is shown below.


| Experiment # | Time elapsed in seconds (stable v2.12.2) | Time elapsed in seconds (patched version) | Page Faults (v2.12.2) | Page Faults (patched) |
|--------------|------------------------------------------|-------------------------------------------|-----------------------|-----------------------|
| 1            | 104.23                                    | 106.36                                     | 7924930                | 8783578                |
| 2            | 103.96                                    | 106.04                              | 7927112                | 8764349                |
| 3            | 104.15                                    | 106.00                                     | 7928092                | 8777599                |
| σ of each set of samples            | 0.13868                                    | 0.19732                                     | 1618.6               | 9840.9                |


Each of the performance tests performed the following tasks representing diffs between <older kernel version> and <newer kernel version> as shown in the testing script above.


## Lessons Learned

A big goal of this project was to be exposed to and contribute to the open source community, using the content from CS3281 to work on a tool that is well tested and commonly used in technical circles. Git perfectly fit the bill and we can confidently say that we earned valuable experience sifting through email threads, man pages, and corresponding with the people who built git from the ground up.

Perhaps the most difficult part of the project was understanding the part of the codebase we touched before adding our own multithreaded functionality.  At one point, in an effort to better visualize the interactions between functions and files in the codebase, Harrison generated a call graph of the codebase that was 40 page PDF. Regardless, we continued breaking down the code piece by piece, became `$ git grep` experts, and ultimately figured out how the code we touched functioned. Still, it was not until after a full day spent sifting through code that we realized that our original plan to parallelize multi-file diffs would not be possible without making sweeping changes throughout the codebase. 

Though some of the concepts learned in CS3281 can offer fantastic performance improvements when used properly in specific use cases, we found out ourselves that program design can greatly influence the possible gains from using these tools. Though parallelizing diffs by file seemed like an awesome opportunity to improve diff performance by a factor of almost 2 or 4 on most machines, the multiprocessing scarcity during the 2005 inception of `diff.c` might have influenced the decision to not parallelize diffs from the start. Now, it is an extremely difficult task.

Moreover, when we began using locks to lock filespecs when we were attempting to check for renames, performance decreased drastically from a lockless model. Although locks can guarantee synchronicity between threads, they come with an additional cost of execution time strongly dependent on the order in which users request them. As parallelization increases and synchronization remains important, maintaining consistency and reaching optimal performance become challenging tasks to balnace.

All in all, we learned an enormous amount about working with gargantuan open-source code bases and were pleased to have the opportunity to contribute valuable information back to the git maintainers to whom we owe so much.

## Division of Labor and Final Thoughts

All coding work was done on Samuel’s machine and done in a pair programming style. Together, we spent several full days decomposing the git codebase and attempting to understand the complexities of diff implementation. After finally having a basic grasp on the part of the code base we added to, we began designing implementation solutions that fit within the design parameters set forth by the existing code. After coding the initial design, Sam spent several extra hours smashing bugs and corresponding with Git maintainers (see testing section for details on that exchange).

All documentation required as part of the project (slides, `design.md`, `finalreport.md`, `README.md`, our project day design slide) were completed on Google docs. We collaboratively worked through our write-up and attempted to construct a concise, accurate depiction of changes made to the codebase. Harrison spent several extra hours altering the documents to ensure that they properly reflected all of the work we put in over the course of the last month or so.

Overall, we had a positive experience working together with a frustratingly large and complex codebase. Although enormous performance increases were not attained by applying our threading models to rename detection, we are still able to provide valuable feedback to the Git community, as these we took a deep dive and examined the last two parts of the codebase they were looking to parallelize.
