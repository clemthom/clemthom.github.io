---
title: "Git submodules"
summary: Working with git submodules
date: 2024-12-22
tags: ["git","submodule"]
author: "Clement Thomas"
---

* Git submodules is a way to keep the content of repository in a sub-directory of another repository.
* `160000` - special mode in git to denote we are recording a commit as directory entry. 

{{< highlight shell >}}

# == add new submodule to git

git submodule add https://github.com/gitorg/repo \
    diff_dir_name_if_required [-b branch]

# == git diff with submodule info

git diff --cached --submodule


# == cloning repo with submodules

git clone https://github.com/gitorg/repo
git submodule init
git submodule update

# or in single command

git clone --recurse-submodules https://github.com/chaconinc/MainProject

# == fetch and checkout nested submodules

git submodule update --init --recursive

# == get new content in the submodule

navigate to submodule directory and do

git fetch # and later
git merge

git diff --submodule

# or

git submodule update --remote [directory_name]

# == git track different branch of submodule

git config -f .gitmodules submodule.DbConnector.branch stable

git submodule update --remote


{{< /highlight >}}

### References

* https://git-scm.com/book/en/v2/Git-Tools-Submodules
