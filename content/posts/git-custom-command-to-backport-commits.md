+++
title = "Git custom command to backport commits"
description = """When backporting commits in git replace cherry picked commit
line with custom message to indicate it is a backported commit."""
date = "2024-01-07T17:49:38+0530"
author = "Mani Kumar"
categories = ["git", "tricks"]
tags = ["git", "sed"]
+++

Problem
-------

Git provides an option `git cherry-pick -x` to record the commit message from
which the cherry picked commit was formed when backporting changes.  This
commit message "(cherry picked from commit ...)" is very generic and does not
convey why cherry pick was done.  Or we may want to make additional changes to
the original commit message.

For this reason most projects will have a custom message e.g. "Backport-Of:
{SHA1}" to be appended for backported changes to indicate from which commit
they were cherry picked from.  To add this custom message we need to open a
text editor each time cherry-pick was performed with `git cherry-pick -xe`
option and manually change the SHA1 hash line.

This is a repeated and exhaustive task if there are large number of commits to
backport.  For this we create a custom command to search and replace the SHA1
hash line with the custom message.

Solution
--------

### Search and replace with `sed`

Searching and replacing cherry picked from commit line can be done by `sed`
stream editor and regular expression as below.

```bash
sed 's/(cherry picked.*\([0-9a-f]\{40\}\))/Backport-Of: \1/'
```

Explanation of above regular expression is simple first we match any line that
starts with "(cherry picked" and then capture the string that is made of
numericals and lowercase alphabets "\([0-9a-f]\{40\}\)" and it is of 40
characters length this is SHA1 hash pattern. This is then available as
parameter 1 in `sed` substitute command.

`sed` substitute command performs the search and replace of cherry picked from
commit line with "Backport-Of: " commit message and replaces parameter 1 with
SHA1 hash.

### Git custom command to automate search and replace

Add above `sed` command to search and replace cherry picked from commit line
to a bash script as below. Give it a sensible name such as `gcpx_ed` and make
it executable and available in `$PATH` e.g. copy it into `~/.local/bin/`.

```bash
$ bat -p `which gcpx_ed`
#!/bin/bash
#
# Replace SHA1 line in git cherry pick commit message

sed -i 's/(cherry picked.*\([0-9a-f]\{40\}\))/Backport-Of: \1/' $1
```

This acts as a simple text editor which opens a file given to it and performs
text substitution in-place and closes it with SUCCESS or ERROR.

Use this as a git editor before executing `git cherry-pick -xe {SHA1}` so that
custom message is automatically added.

```bash
GIT_EDITOR=gcpx_ed git cherry-pick -xe {SHA1}
```

To simplify this command create an alias as below and save it in `~/.bashrc`
or `~/.zshrc`.

```bash
gcpx='GIT_EDITOR=gcpx_ed git cherry-pick -xe'
```

Now we have the git custom command `gcpx` to backport commits with custom
message cherry picked from commit line. It can be invoked as `gcpx {SHA1}`.
