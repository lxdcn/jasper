---
layout: post
cover: assets/images/cover-gitw.png
subclass: post
title: "Git wrap script to ignore files that have already been committed"
navigation: true
logo: assets/images/host.png
date: 2016-01-13 22:56:56
categories: lxd
tags: tech shell misc en
comments: true
---

### Motivation

As a git user, given this sceniro:

> You made certain changes to some files but you don't want to commit it, nor do you want see them under `git status -s` or `git diff` etc.

In other words:

> You want git to ignore certain files but unfortunately they have already been tracked.

what you gonna do?


I googled around a lot, all I can get is remove these files entirely from repo by run `git rm --cached <file>` (so `.gitignore` will take effect), but it's not always doable, because your teammates may still need these files exist in repo.

Then I checked git hooks, hoping I could write some scripts to preserve these local changes before execute `git status` and `git diff`. Unfortunately I failed again, what git hooks can do is pretty limited, furthermore I realized not only do I need to 'deceive' `git status` and `git diff`, `git add` `git checkout` `git pull` and a lot other commands also require the same trick.

Up to this point I had very little choices, either I figure out a way to make `.gitignore` also ignore tracked files (git porcelain command `git check-ignore` has a switch to do this), or write a script wrap git entirely. I chose the latter.


### Usage


The code is [here](https://gist.github.com/lxdcn/c6ab6365bdde315e4722), you are very welcome to enhance or report issues, I'll illustrate this script line by line in next section, but at first let's see the usage.

- Download the script and put under $PATH directory and make it executable, make sure can run `gitw` directly from command line
- You may have aliased common git command to a shorter command (e.g. `alias gpr='git pull --rebase'`) or shell plugins have done the same thing for you, in that case you need to put `alias git='gitw'` in your shell initialization file. Also you probably need to give `gitw` the same command line completion as original git, so put `compdef gitw=git` in shell init file too.
- Add still-want-ignore-even-tracked files to `.gitignore` per git working directory, most likely you need to add .gitignore to `.gitignore` as well


### Explanation


{% highlight bash %}
if [[ ! -d '.git' ]]; then
	\git "$@"
	exit $?
fi

mkdir -p .git/w
{% endhighlight %}

This script should not work on `git clone` or `git init`, so it will pass command arguments directly to original git then exit if we are not under a git working directory.

{% highlight bash %}
mkdir -p .git/w
{% endhighlight %}

This script preserves local change to a directory named `w` under `.git`, which will be created here if not exists.


{% highlight bash %}
original_command=`\git "$1" -h 2>&1 | head -n 1 | awk '{print $3}'`
{% endhighlight %}

In following steps we need to pattern match git command to decide whether we wrap specific git command or not, but you probably already alias git command to a shorter name (eg. `co = checkout`, `di = diff`). We extract original git command by parsing `git <command_or_alias> -h`.


{% highlight bash %}
function ignored_dirty_index_func() {
	for dirty_file in `\git status --porcelain | awk '{print $2}'`; do
		\git check-ignore -q --no-index $dirty_file
		if [[ $? -eq 0 ]]; then
		  echo $dirty_file
		fi
	done
}
ignored_dirty_index=`ignored_dirty_index_func`
{% endhighlight %}

This function here identifies local changed files need to be ignored. We use porcelain version<sup>[[0](http://git-scm.com/book/en/Git-Internals-Plumbing-and-Porcelain)]</sup> of `git status` (to list all local changed files) cross check with `git check-ignore` (to check if each file is ignored) to get a list of files with local changes but we'd like to ignore.

Pay attention to `git check-ignore` has a switch `--no-index`, meaning not look in the index when undertaking ignore checks. This is exactly what we need for `.gitignore` but only exists in `git check-ignore`<sup>[[1](http://git-scm.com/docs/git-check-ignore.html)]</sup>.


{% highlight bash %}
echo "$original_command" | grep '^\(add\|status\|checkout\|pull\|rebase\|merge\|diff\|stash\|reset\|commit\)$' > /dev/null
{% endhighlight %}

This script only works on these git commands.


{% highlight bash %}
if [[ $? -eq 0 ]] && [[ -n "$ignored_dirty_index" ]]; then
  echo 'gitw is helping...'
	echo "$ignored_dirty_index" | rsync -R --files-from - . .git/w
	echo "$ignored_dirty_index" | \git checkout-index -f --stdin
	\git "$@"
	echo "$ignored_dirty_index" | rsync -R --files-from - .git/w .
else
	\git "$@"
fi
{% endhighlight %}

If the current git command lies in commands we should wrap and there are local changes need to be ignored, this script should do the job. At first we copy all the target files to `.git/w` with directory structure perserved, then drop local changes for these files before execute original git, finally we recover local changes by copy target files from `.git/w` to workspace.



References:

- [0] http://git-scm.com/book/en/Git-Internals-Plumbing-and-Porcelain
- [1] http://git-scm.com/docs/git-check-ignore.html




<br />
### Update 2016-01-31 22:22:22:
Thanks to teammate [@SuXiaoKai](https://github.com/SuXiaoKai), `git update-index --assume-unchanged <path>` can do this job pretty well.

