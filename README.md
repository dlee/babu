# Babu: sysadmin automation in Bash

[Babu](https://github.com/dlee/babu) is a tiny automation script written in the venerable Bash scripting language. It is heavily influenced by [babushka](https://babushka.me/) and follows the same strategy for automating computing chores:
> for each dependency (dep) of the job to do, a test, and the code to make that test pass -- test-driven sysadmin."

The reason for creating a separate project was because babushka required ruby to run, but I wanted something that could run on ruby-less systems. [Installation](#installation) only requires bash, which is found on virtually every *nix system since the past decade.

This project's name obviously comes from babushka. It was originally intended to be `babu.sh` for extra cleverness, but convenience and ease favored `babu`.

Babushka is an [old woman](https://en.wikipedia.org/wiki/Babushka); babu is an [old man](https://en.wikipedia.org/wiki/Babu_(title)).

## Usage

A dep (standing for "dependency") is the core unit of a task in babu. Each dep is named and can be invoked by running `babu $dep_name`:
```
$ babu "install zsh"
```
The above command will run the dep named "install zsh" that's defined in the `Babufile` in the current directory. If no argument is provided to `babu`, `babu` will run the dep named "default".
```
$ babu
```
is the same as
```
$ babu default
```

If you need to pass additional information into the script, you can do so with [environment variables](#variables):
```
$ EXTRA_VAR1=hello EXTRA_VAR2=world babu
```

### Babufile

Babu provides a DSL for defining deps in Bash. A dep mainly consists of `met` and `meet` functions; the former checks for a condition (and returns the result as a return code) and the latter satisfies it. These functions are written in pure Bash and can run any command.

A `Babufile` is simply a Bash script that defines dependencies for `babu` using this DSL:

```bash
# Babufile
dep "on staging branch"
{
  met() {
    branch=`git symbolic-ref --short HEAD`
    echo "Currently on $branch."
    [ "staging" == "$branch" ]
  }
  meet() {
    git checkout staging
  }
}
```

A `dep $dep_name` starts the definition of a dependency. The first argument to `dep` will be the unique name of the dep (e.g. "on git branch" in above example) and will be used to run the particular dep or make the dep a requirement of another dep (See [requires](#requires)). Once a `dep` is declared, the following function definitions for `met()` and `meet()` will belong to that dep:
* `met()` tests for whether or not the dependency is met. A `0` return code (in the shell, 0 means success) will be considered as the dependency being met.
* `meet()` hosts code that satisfies the dependency in case `met()` returns a non-`0` return code.

```
$ git checkout master
$ babu "on staging branch"
on staging branch {
  Currently on master.
  meet {
    Switched to branch 'staging'
    Your branch is up-to-date with 'origin/staging'.
  }
  Currently on staging.
} ✓ on staging branch
```
`babu` ran the `met()` function to see if the current branch was `staging`, and since it was not, ran `meet()` to fulfill the dependency. Notice how the `met()` check was run again after `meet()` was run. If the second `met()` check had failed, `babue` would have aborted with an error. However, in the above example, we can see that it has correctly satisfied the dependency. Now if we run the `babu` command again:

```
$ babu "on staging branch"
on staging branch {
  Currently on staging.
} ✓ on staging branch
```
Since the current branch is already `staging`, `meet()` is skipped.

### Variables
You can pass in variables to `babu` as you would any Bash script, by using environment variables:
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

We've set up a default value for `$TARGET_BRANCH` in the Babufile, but if we pass in an initial value for `$TARGET_BRANCH`, Bash will use the value passed in:
```
$ TARGET_BRANCH=staging babu "on git branch"
on staging branch {
  Currently on staging.
} ✓ on staging branch
```

If we don't pass in an initial value for `$TARGET_BRANCH`, then the default value of `master` will be used:
```
$ babu "on git branch"
on git branch {
  Currently on staging.
  meet {
    Switched to branch 'master'
    Your branch is up-to-date with 'origin/master'.
  }
  Currently on master.
} ✓ on git branch
```

### Requires
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

In the example above, the "default" dependency now requires the "on git branch" dependency. This means that the "on git branch" dependency must be met before the "default" dependency:
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

#### Caveats
Since babu internally converts all deps into Bash functions, there is a lossy dep-to-function name conversion that may cause collision in dep naming: `"babu me good"`, `"babu.me.good"`, and `"babu-me-good"` would all be mapped to the same function name. The rule of thumb is to avoid dep names whose only difference is non-alphanumeric characters.

## Installation

`babu` is a self-contained single-file script that should run on most Unix systems. `babu` was created out of a desire to have babushka-like automation script with minimal dependencies (babushka requires ruby). The only requirement for `babu` is that you have bash version 3 or above.

```
$ git clone https://github.com/dlee/babu.git
$ cp babu/babu /bin/babu
$ chmod a+x /bin/babu
```
