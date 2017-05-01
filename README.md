# Parallelizing Git Rename Detection

Samuel Lijin and Harrison Stall: an experiment in parallelizing git rename detection.

Note: our project files can be found under the `docs` directory and inside `slides.pdf`.

## Overview

If a file is renamed between commits in git, that information is not recorded in the git database; when generating changelogs, git decides what changes between commits constitute renames in realtime.

For example, if I have a git repository with `file1.c` and make the following commits:

* `COMMIT 1` add a line to `file1.c`
* `COMMIT 2` rename the file to `file2.c` and delete a line
* `COMMIT 3` change a line in `file2.c`

Diffing `COMMIT 1` and `HEAD` will show that `file1.c` and `file2.c` are really the same file, and it will show all 3 sets of changes in said diff. Internally, this is done by comparing all potential rename sources and rename destinations, and declaring the most likely to be renames.

These computations are currently performed sequentially; however, it should be possible to perform many of these computations in parallel. Our project modifies this operation by partitioning these computations amongst a thread pool.

## Building the Project

To build git:
~~~
$ git clone https://github.com/CS3281-vu/git.git
$ cd git
$ git checkout cs3281.final.project
$ make
~~~

To run the git test suite:
~~~
make test
~~~

### Performance Testing

One benchmark we used was running diffs between all consecutive tagged versions of the Linux kernel. To reproduce this, save the following script:

~~~
#!/bin/sh

if test -z "$GIT_BINARY"
then
    echo "GIT_BINARY must be set" 1>&2
    exit 1
fi

prev_tag=
$GIT_BINARY tag | grep -v - -- | sort -V | while read tag
do
    if test -n "$prev_tag"
    then
        echo "Diffing $prev_tag $tag"
        $GIT_BINARY --no-pager diff --raw -M -l0 $prev_tag $tag >/dev/null
    fi
    prev_tag=$tag
done
~~~

and invoke it as follows:
~~~
$ GIT_BINARY=/path/to/git /usr/bin/time -v /path/to/script.sh
~~~

By running the script with `GIT_BINARY=/usr/bin/git` and `GIT_BINARY=/path/to/custom/git`, one can get an idea of how our version of git performs relative to the version of git distributed with one's Linux distro.

