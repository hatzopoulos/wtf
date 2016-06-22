  * [Whitespace Total Fixer](#whitespace-total-fixer)
    * [Why you should use it](#why-you-should-use-it)
    * [Quick Install](#quick-install)
    * [How to use it](#how-to-use-it)
    * [Git `pre-commit` hooks](#git-pre-commit-hooks)
      * [Careful version](#careful-version)
      * [Simple version](#simple-version)
  * [Exciting origin story](#exciting-origin-story)
  * [Whitespace issues addressed](#whitespace-issues-addressed)
  * [Reporting](#reporting)
  * [Todo](#todo)
  * [Bugs](#bugs)
  * [Author](#author)
  * [License](#license)

Whitespace Total Fixer
======================

Identifies and/or fixes inconsistent whitespace and line endings in
text files, so that they don't clog up your commits to version control
systems like Git, Mercurial, or Subversion.

### Why you should use it:

* It's like an incomprehensible shell-script one-liner (e.g. `sed -e 's/LINENOISE/'`), but way better
* It's similar to [`git
  stripspace`](https://www.kernel.org/pub/software/scm/git/docs/git-stripspace.html),
  but more flexible and detailed.
* `wtf.py` is a simple Python2 script (tested with 2.7.5) with *no
  dependencies beyond the standard Python library*.


### Quick Install
```
curl https://raw.githubusercontent.com/dlenski/wtf/master/wtf.py > ~/bin/wtf.py && chmod 0755 !#:3
```

### How to use it

[See below](#options) for options to control exactly which
whitespace issues it fixes, but here are some examples:

```bash
# consistent whitespace from programs that generate text files
dump_database_schema | wtf.py -o clean_output.sql

# in-place editing
wtf.py -i file1.txt file2.txt file3.txt
wtf.py -I.bak file1.txt file2.txt file3.txt # ditto, with backups

# summarize a bunch of files without actually editing them (-0)
find . -name "*.txt" -exec wtf.py -0 {} \;

# more advanced: find interesting text files. pass options and a directory
wtf-find() {
    find "${@: -1}" -not \( -name .svn -prune -o -name .git -prune \
         -o -name .hg -prune \) -type f -exec bash -c \
         'grep -Il "" "$1" &>/dev/null && wtf.py '"${*:1:$#-1}"' "$1"' _ {} \;
}
wtf-find -0 .

# exit status
wtf.py file1.txt file2.txt file3.txt > /dev/null
if (( $? == 10 )); then
  echo "issues fixed"
elif (( $? == 20 )); then
  echo "unfixed issues!"
fi
```

Git `pre-commit` hooks
----------------------

You can use [Git `pre-commit` hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
to automatically run `wtf` and cleanup whitespace in your repository
prior to every commit.
(You can bypass the hook, however, with `git commit -n`.)

Create the file `.git/hooks/pre-commit` in your repository, and ensure that it is executable
(`chmod +x .git/hooks/pre-commit`).

### Careful version

This version only modifies your commits, and does not touch Git's
working tree.  If you understand and use Git's index ("staging area"),
you should use this version. It will play nicely with `git add --patch`.

Note that when a file changes in your commit but remains unchanged in your working tree, the file will be marked `modified` by `git status` immediately after the commit.

[This version](https://github.com/dlenski/wtf/blob/HEAD/pre-commit.careful)
will run `wtf`, with the default options, on all the to-be-committed
text files.

### Simple version

This version modifies Git's working tree as well as your commits. **This version will not play nicely with
`git add --patch`.**

This version will run `wtf -i`, with any other options you add to `wtf_options`, on all the
to-be-committed text files. They will be cleaned up in the commit, as
well as in your working tree.

```bash
#!/bin/sh
wtf_options=''

# get a list of to-be-committed filepaths, EXCLUDING files considered
# by Git to be binary
committees=$(git diff --cached --numstat --diff-filter=ACMRTU|egrep -v ^-|cut -f3-)

# Run Whitespace Total Fixer in-place, and re-add files modified by it
for committee in $committees
do
	wtf -i $wtf_options "$committee" || git add "$committee"
done
```

Exciting origin story
---------------------

One day at work, I spent way too much time wrangling commits
laden with whitespace issues generated by recalcitrant text-editing
tools on multiple platforms.

That evening, I went home and spent way too much time writing this
program instead!

<a name="options"/>Whitespace issues addressed
----------------------------------------------

WTF currently fixes, or simply reports, a few common types of whitespace
issues. Most of these issues offer three possible command-line options
enabling the user to fix, report, or ignore the issue.

* Remove trailing spaces at the ends of lines (default is **fix**):

        -t, --trail-space
        -T, --report-trail-space
        -It, --ignore-trail-space

* Remove blank lines at the ends of files (default is **fix**):

        -b, --eof-blanks
        -B, --report-eof-blanks
        -Ib, --ignore-eof-blanks

* Ensure that a new-line character appears at the end of the file (default is **fix**):

        -n, --eof-newl
        -N, --report-eof-newl
        -In, --ignore-eof-newl

* Make sure that all lines have matching EOL markers (that is, no
  mixing of lf/crlf). Default is to **fix** non-matching EOL
  characters by making them all the same as the first line of the
  file. The desired EOL markers can also be set to a specific value
  (`lf`/`crlf`/`native`), in which case all lines will unconditionally
  receive this marker.

        -E ENDING, --coerce-eol ENDING
        -e ENDING, --expect-eol ENDING
        -Ie, --ignore-eol

* Check for spaces followed by tabs in the whitespace at the beginning
  of a line; fixing this condition requires setting either
  `--change-tabs` or `--change-spaces`. The default is to report and
  **warn**:

        -s, --tab-space-mix
        -S, --report-tab-space-mix
        -Is, --ignore-tab-space-mix

* Change tabs in the whitespace at the beginning of a line to the
  specified number of spaces (default is not to change tabs at
  all). This option will *not* touch leading tabs when they are mixed
  with spaces, *unless* `--tab-space-mix` is also specified.

        -x SPACES, --change-tabs SPACES

* Change the specified number of consecutive spaces in the whitespace
  at the beginning of a line to tabs (default is not to change spaces
  at all). This option will *not* touch leading tabs when they are mixed
  with spaces, *unless* `--tab-space-mix` is also specified.

        -y SPACES, --change-spaces SPACES

Reporting
---------

Unless the `-q`/`--quiet` option is used, WTF will summarize each file
processed in which any whitespace issues were found and/or fixed. With
`-v` it will also report issue-free files.

    $ wtf -0 nightmare.txt    # -0 is equivalent to > /dev/null
    nightmare.txt LINE 8: WARNING: spaces followed by tabs in whitespace at beginning of line
    nightmare.txt:
        CHOPPED 1 lines with trailing space
        CHOPPED 0 blank lines at EOF
        ADDED newline at EOF
        CHANGED 1 line endings which didn't match crlf from first line
        WARNED ABOUT 1 lines with tabs/spaces mix

WTF will return the following exit codes on successful operation:

* `0`: no issues seen (or `-X`/`--no-exit-codes` specified)
* `10`: issues fixed
* `20`: unfixed issues seen

Todo
----

* Stability tests?
* Unicode tests?

Bugs
----
Corrupts source code files written in the [Whitespace programming language](https://en.wikipedia.org/wiki/Whitespace_(programming_language)).

Anything else?

Author
------
&copy; Daniel Lenski <<dlenski@gmail.com>> (2014-2016)

License
-------
[GPL v3 or later](http://www.gnu.org/copyleft/gpl.html)
