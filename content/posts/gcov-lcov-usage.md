+++
title = "Gcov with Lcov usage"
description = """How to use gcov with lcov frontend to generate code coverage \
reports for C++ projects."""
draft = false
date = "2021-06-16"
author = "Mani Kumar"
categories = ["notes", "code", "gcov"]
tags = ["gcov", "lcov", "coverage"]
+++

Gcov
----

Gcov is a code coverage tool used in concert with GCC to analyze the programs
and create more efficient, faster running code. Also discover the untested
parts of a program. It uses the static library 'libgcov' which should be
linked to the program binary.

Lcov
----

Lcov is a graphical frontend for Gcov. As gcov produces text based report
which is not very intuitive on its own. Using lcov tool we can get better
report and also convert it to html format so we can view it in web browsers.
Lcov is part of Linux test project which is used to provide code coverage
reports for linux kernel and many other open source projects.

Steps to generate code coverage report
--------------------------------------

Step 1: Compile with '--coverage' option to enable linking to gcov.

```sh
# --- In build server ---
# Compile with '-g --coverage' options, optionally use '-pg' for gprof symbols
# On successful compilation this step produces '.gcno' (aka. gcov counter)
# file for every source file i.e. '.cpp' file compiled with '--coverage'
# option.
$ GCOV_COMPILE_OPTIONS="--coverage -pg -g"
$ make -j12 \
    CFLAGS="$GCOV_COMPILE_OPTIONS" \
    CPPFLAGS="$GCOV_COMPILE_OPTIONS" \
    CXXFLAGS="$GCOV_COMPILE_OPTIONS" \
    LDFLAGS="$GCOV_COMPILE_OPTIONS"
```

Step 2: Generate the baseline coverage data file.

```sh
# Generate the baseline coverage data file i.e. initial zero coverage report
# use this to compare with test case coverage report to get accurate
# percentage even when not all source code files were loaded during the
# test/s.
# TIP: use '--no-external' to remove coverage data for external source
# files not provided by '--directory' option.
$ lcov --capture --initial --directory . \
  --output-file a_out_lcov_base.info
```

Step 3: Run the test case.

```sh
# --- In test server ---
# When the program is not run in the same place it was compiled,
# set these environment variables to generate '.gcda' files
# refer:
# https://gcc.gnu.org/onlinedocs/gcc/Cross-profiling.html#Cross-profiling
# We can ignore this step if the program is run in same place where it was
# compiled.
$ export GCOV_PREFIX=$PWD
$ export GCOV_PREFIX_STRIP=3

# run the binary compiled with gcov flags
# When the process exits cleanly then the '.gcda' (aka. gcov data) files will
# be generated. These files correspond to the '.gcno' files in compilation
# step.
$ ./a.out

# If '.gcda' files are generated then copy them back to build server.
$ find . -regex ".*/.*\.\(gcda\)" -exec tar -rvf a_out_gcov_v0.tar {} \;

# --- In build server ---
$ scp <test_server_ip>:<path_to>/a_out_gcov_v0.tar
$ cd <a.out_path>
# untar and create *.gcda files in same locations as *.gcno files.
# NOTE: If any '.gcda' file is missing or not found in the same path as the
# '.gcno' and '.cpp' files then the code coverage for that file will be
# ignored.
$ tar -xvf <path>a_out_gcov_v0.tar
```

Step 4: Generate lcov data (aka. tracefile) file for the test case.

```sh
# generate test case coverage data file
$ lcov --capture --directory . --output-file a_out_lcov_test.info
```

NOTE: We can run as many test cases as we want, generate the lcov data file
for each test case and add them in this step to combine and create the final
code coverage report.

Step 5: Combine all the lcov data files of all the test cases.

```sh
# combine lcov before and after tracefiles i.e., base & test lcov files
$ lcov --add-tracefile a_out_lcov_base.info \
    --add-tracefile a_out_lcov_test.info \
    --output-file a_out_lcov_total.info
```

Step 6: Generate html report using 'genhtml' using the combined lcov
tracefile.

```sh
# convert lcov report to html files
$ genhtml a_out_lcov_total.info \
    --output-directory a_out_lcov_html \
    --demangle-cpp --legend --title "Cpp basic gcov test"
```

Pitfalls:

* The '.gcda' (aka. gcov data) files will be generated only when the
  process exits cleanly and should not be killed or interrupted.
  This is not always desired as in case of servers the processes do not exit.
* The process should have the required permissions to create the '.gcda' files
  in the same file paths as '.gcno' files after considering the effect of
  'GCOV_PREFIX' and 'GCOV_PREFIX_STRIP' environment variables.
  The environment variables may not be passed to process if it is under
  the control of a parent process such as 'systemd'.

To overcome these pitfalls we need to create a wrapper test code, pass the
environment variables at the beginning for the process and call
'__gcov_flush()' or '__gcov_dump()' to dump the gcov data collected upto that
point. Now the process doesn't need to exit to produce the gcov data files.

To make the binary dump gcov data on demand it is best to use a signal handler
and control using external process to send signals such as 'kill' command.
This is further explained with a sample code in below hack.

HACK: dump gcov data on demand
------------------------------

Using signal handlers:

We use signal handler in our wrapper code so we can control gcov data files
generation after our tests are successful and we are ready to take gcov data.

```cpp
/* Create test_main.cpp identical to main.cpp. */

// include signal handler headers
#include <errno.h>
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <stdlib.h>

// implemented in libgcov.a
extern "C" void __gcov_flush(void);

/* handle signal, dump gcov and exit. */
void sig_handler(int sig) {
  switch(sig){
    case SIGINT:
    case SIGTERM:
    case SIGUSR1:
      printf("received signal=[%d]\n", sig);
      break;
    default:
      printf("received unexpected signal=[%d]!!!\n", sig);
      return;
  }
  const char *gcov_prefix = getenv("GCOV_PREFIX");
  const char *gcov_prefix_strip = getenv("GCOV_PREFIX_STRIP");
  printf("printing environment variables, "
          "GCOV_PREFIX=[%s]; GCOV_PREFIX_STRIP=%s;\n",
          gcov_prefix ? gcov_prefix : "",
          gcov_prefix_strip ? gcov_prefix_strip : "");
  printf("dumping gcov data files i.e. .gcda files\n");
  __gcov_flush();
  //printf("exiting process\n"); // exiting is not needed
  //exit(sig);
}

/* test code main func
 *
 * created specifically to add signal handlers and dump gcov data files
 * as gcov generates data files only on process exit.*/
int main(int argc, char **argv) {
  int ret = EXIT_SUCCESS;
  signal(SIGINT, sig_handler); // Register SIGINT
  signal(SIGTERM, sig_handler); // Register SIGTERM
  signal(SIGUSR1, sig_handler); // Register SIGUSR1
  /* NOTE:
   * Calling setenv() within the test process is better than setting
     environment variables in shell as it localizes the process environment
     and works well when the process is started by a fork() & exec().
   * In case of fork() and exec() the environment variables will be same as
     they are in parent process which may not be under our control.
   * Environment variables are copied from the shell/parent process when
     the process is started and cannot be changed/added from outside it. */
  setenv("GCOV_PREFIX", get_current_dir_name(), 1);
  setenv("GCOV_PREFIX_STRIP", "6", 1);
  // NOTE: add a.out/Cpp binary logic here
  ret = execute(argc, argv);
  return ret;
}
```

Debug gcov data file generation
-------------------------------

When gcov data files '.gcda' files are not generated or when the coverage
report is not as expected then use these steps to debug the root cause.

```sh
# --- In test server ---
# find all gcov files in pwd
$ find . -regex ".*/.*\.\(gcno\|gcda\)" -type f

# set/echo/unset gcov run time environment variables for cross profiling
$ export GCOV_PREFIX=$PWD; export GCOV_PREFIX_STRIP=6;
$ echo $GCOV_PREFIX; echo $GCOV_PREFIX_STRIP
$ unset GCOV_PREFIX; unset GCOV_PREFIX_STRIP

# send signals to process that can be caught in the process
$ kill -s SIGINT `pidof ./a.out` # sig = 2
$ kill -s SIGTERM `pidof ./a.out` # sig = 15
$ kill -s SIGUSR1 `pidof ./a.out` # sig = 10

# check logs & core files
```

Fine tune lcov report
---------------------

* Exclude external source files, filter and fine tune code coverage report.
* Add test case descriptions using 'gendesc'.

References
----------

* [LCOV Code Coverage][1]
* [HOWTO: Dumping gcov data at runtime - simple example][2]
* [Using the GNU Compiler Collection (GCC)][3]
* [Testing Linux, one syscall at a time.][4]
* `man lcov(1), gcov(1), genhtml(1)`

[1]: https://wiki.documentfoundation.org/Development/Lcov
[2]: https://www.osadl.org/Dumping-gcov-data-at-runtime-simple-ex.online-coverage-analysis.0.html
[3]: https://gcc.gnu.org/onlinedocs/gcc/index.html
[4]: https://linux-test-project.github.io/
