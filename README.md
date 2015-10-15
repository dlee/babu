# Babu: sysadmin automation in Bash

[Babu](https://github.com/dlee/babu) is a tiny automation script written in the venerable Bash scripting language. It is heavily influenced by [babushka](https://babushka.me/) and follows the same principle for automating computing chores:
> for each dependency (dep) of the job to do, a test, and the code to make that test pass -- test-driven sysadmin."

This project's name obviously comes from babushka. It was originally intended to be `babu.sh` for extra cleverness, but convenience and ease favored `babu`.

Babushka is an [old woman](https://en.wikipedia.org/wiki/Babushka); babu is an [old man](https://en.wikipedia.org/wiki/Babu_(title)).

## Usage

Running `babu $dependency` will run the named dependency defined in a `Babufile` in the current directory. If no `$dependency` is provided as an argument to `babu`, `babu` will run the dependency named "default".

```
$ babu
```
is the same as
```
$ babu default
```

If you need to pass additional information into the script, you can do so with environment variables:
```
$ EXTRA_VAR1=hello EXTRA_VAR2=world babu
```

### Babufile

A `Babufile` is simply a Bash script that defines dependencies for `babu` using its special `dep` declaration convention:

```bash
# Babufile

: ${TARGET_BRANCH:="master"} # $TARGET_BRANCH defaults to "master"

dep "on git branch"
{
  met() {
    branch=`git symbolic-ref --short HEAD`
    echo "Currently on $branch."
    [ "$TARGET_BRANCH" == "$branch" ]
  }
  meet() {
    git checkout "$TARGET_BRANCH"
  }
}
```

A `dep` starts the definition of a dependency. A dependency is named (e.g. "on git branch" in above example) and can be required by other dependencies. `dep` can be followed by function definitions for `met()` and `meet()`:
* `met()` tests for whether or not the dependency is met. A `0` return code will be considered as the dependency being met.
* `meet()` hosts code that satisfies the dependency in case `met()` returns a non-`0` return code.

```
$ git checkout master
$ TARGET_BRANCH=staging babu "on git branch"
on git branch {
  Currently on master.
  meet {
    Switched to branch 'staging'
    Your branch is up-to-date with 'origin/staging'.
  }
  Currently on staging.
} ✓ on git branch
```
`babu` ran the `met()` function to see if the current branch was `staging`, and since it was not, ran `meet()` to fulfill the dependency. Notice how the `met()` check was run again after `meet()` was run. If the second `met()` check had failed, `babue` would have aborted with an error. However, in the above example, we can see that it has correctly satisfied the dependency. Now if we run the `babu` command again:

```
$ git checkout master
$ TARGET_BRANCH=staging babu "on git branch"
on git branch {
  Currently on staging.
} ✓ on git branch
```
Since the current branch is already `staging`, `meet()` is skipped.

Dependencies can require other dependencies by using the `requires` directive:

```bash
# Babufile

: ${TARGET_BRANCH:="master"} # $TARGET_BRANCH defaults to "master"

dep "git"
{
  met() {
    which -s git
  }
  meet() {
    brew install git
  }
}

dep "on git branch"
{
  requires "git"
  met() {
    branch=`git symbolic-ref --short HEAD`
    echo "Currently on $branch."
    [ "$TARGET_BRANCH" == "$branch" ]
  }
  meet() {
    git checkout "$TARGET_BRANCH"
  }
}

dep "default"
{
  requires "on git branch"
  met() {
    echo "We're good"
  }
  meet() {
    echo "Nothing to do"
  }
}
```

In the example above, the "default" dependency now requires the "on git branch" dependency. This means that the "on git branch" dependency must be met before "default" dependency:
```
$ babu
default {
  on git branch {
    git {
    } ✓ git
    Currently on staging.
    meet {
      Switched to branch 'master'
      Your branch is up-to-date with 'origin/master'.
    }
    Currently on master.
  } ✓ on git branch
  We're good
} ✓ default
```

Notice how the "on git branch" dependency was satisfied before we even ran the `met()` for "default". Furthermore, the "git" dependency was satisfied before we checked the "on git branch" dependency. Requires can thus be nested.

You can also provide multiple requires by calling `requires` multiple times:
```bash
dep "needy task"
{
  requires "need 1"
  requires "need 2"
  requires "need 3"
}
```

## Installation

`babu` is a self-contained single-file script that should run on most Unix systems. `babu` was created out of a desire to have babushka-like automation script with minimal dependencies (babushka requires ruby). The only requirement for `babu` is that you have bash version 3 or above.

```
$ git clone git@github.com:dlee/babu.git
$ cp babu/babu /bin/babu
$ chmod a+x /bin/babu
```
