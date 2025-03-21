+++
title = "GTags usage"
description = "Gtags & Global usage doc for tagging and browsing cpp projects."
draft = false
date = "2021-04-24"
author = "Mani Kumar"
categories = ["usage", "gtags", "cpp"]
tags = ["gtags", "cpp"]
+++

> GNU Global is a source code tagging system that works the same way across
> diverse environments. It is similar to `ctags` or `etags`, but is different
> from them at the point of independence of any editor.
>
> -- GNU Global official [tutorial][li_1]

In this post we record the simplified and customized steps for a large Cpp
project. To know about GNU Global tool and its complete documentation please
refer to the official tutorial. A link to this tutorial is provided in the
references section below.

Basic usage
-----------

For a small project it is easy to build the `gtags` database. Just go into
the project directory and execute `gtags` command.

```sh
cd CPP_PROJ1/src/
gtags
```

It generates three database files as shown below.

```sh
ls G*
  GPATH  GRTAGS  GTAGS
```

* GTAGS - definition database
* GRTAGS - reference database
* GPATH - path name database

Advanced usage
--------------

In a large project it is better to fine tune the project files to index for
fast and accurate database results.

### Select files to index

Select the list of files to index by providing the file paths in a text file
to `gtags` similar to `cscope` file list.

```sh
# Generate list of files to index for gtags/cscope
PROJ_SRC_DIR=$PWD
PROJ_BUILD_DIR=$PROJ_SRC_DIR/build

# find all source files ignore everything else
find $PROJ_SRC_DIR \
    -path "$PROJ_SRC_DIR/CodeCoverage/*" \
    -path "$PROJ_SRC_DIR/StructuredData/*" \
    -path "$PROJ_SRC_DIR/UnitTesting/*" \
    -prune -o -path "$PROJ_SRC_DIR/autom4te.cache/*" \
    -prune -o -path "$PROJ_SRC_DIR/build/*" \
    -prune -o -path "$PROJ_SRC_DIR/build-aux/*" \
    -prune -o -path "$PROJ_SRC_DIR/m4/*" \
    -prune -o -regex '.*/.*\.\(cc\|hh\)$' \
    -print >$PROJ_BUILD_DIR/proj_src_files.txt

# Review and edit the generated project source files as suited.
# Generate GNU Global database files
gtags -f $PROJ_BUILD_DIR/proj_src_files.txt
```

### Separate Gtags DB path

If the source files are on a read-only device, such as CDROM or we just want
to keep the gtags database files in another path then we can make the tag
files in that path using `GTAGSROOT` environment variable.

```sh
# make db target path
PROJ_SRC_DB_DIR=/var/dbpath/
mkdir $PROJ_SRC_DB_DIR

# cd into project src directory as previous
cd $PROJ_SRC_DIR

# generate gtags database
gtags -f $PROJ_BUILD_DIR/proj_src_files.txt $PROJ_SRC_DB_DIR

# export GTAGS env variables
export GTAGSROOT=$PROJ_SRC_DIR
export GTAGSDBPATH=$PROJ_SRC_DB_DIR
```

### Index non-src symbols

When we want to index symbols that are not defined in the project source tree
such as the system and third party libraries, headers, macro definitions, then
we can specify library directories with `GTAGSLIBPATH` environment variable.

But this approach requires executing `gtags` command in each directory in the
`GTAGSLIBPATH`. So a better approach is to create symlinks to system
headers and source files in our project directories to make gtags treat them
as part of the project source files.

```sh
ln -s /usr/src/lib .
ln -s /usr/src/sys .
gtags
```

While this is still not a good solution as it clutters the source tree we may
not need this info in real time. Based on my experience we rarely look for
system headers and symbols. I will make a note of all the limitations and
improvements for GNU Global tool and look for near perfect tool that helps us
in understanding Cpp source code.

### Incremental updates

To speed up indexing after each modification to source files we can update the
`GTAGS` database using `global` update command e.g. `global -vu`.

### Force .h files as Cpp files

By default gnu global treats `.h` files as C source files. This will result in
indexing Cpp code incorrectly. To force gnu global to index `.h` files as
Cpp source files set `GTAGSFORCECPP` environment variable to any value. Then
gnu global will treat each file whose suffix is `.h` as a Cpp source file.

Global command usage
--------------------

Once the `GTAGS` database is built we can browse the source code using the
`global` command.

```sh
global func1 # locate definition
global -r func2 # locate references
global 'func[1-3]' # locate references using POSIX regex
global -x func2 # references with details
global -xs X # locate symbols which are not defined in GTAGS i.e. #ifdef X
global -xg '#ifdef' # locate lines which the pattern, similar to egrep
global -P Pattern1 # locate path names which have pattern "Pattern1"
global -f DIR2/fileC.c # print the list of tags in the file
global -xl func[1-3] # search only under the current directory
```

Similar tools
-------------

* [Universal Ctags][2]
* [Cscope][3]
* [OpenGrok][4]

References
----------

For more information and examples refer to below links.

* GNU Global official [tutorial][li_1]

[li_1]: https://www.gnu.org/software/global/globaldoc_toc.html
[2]: https://ctags.io/
[3]: https://cscope.sourceforge.net/
[4]: https://oracle.github.io/opengrok/
