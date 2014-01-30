---
layout: post
title: "Git 实用技巧"
description: "Git 实用技巧"
category: "Git"
tags: []

---

1. git rm, git clean, rm 在版本控制中的差异

    首先清楚 git 工作流中的**working-tree**(工作目录)的概念,  **working-tree** 一般是和 *.git* 目录同级的工作目录，
    也是我们平时要提交的差异文件所在目录，**working-tree** 在远程纯仓库(bare repository)上是不存在的；

    1. `git rm` 从 **index** 和 **working-tree** 中删除，这个过程可回退;
    2. `git rm --cached` 只从 **index** 中移除，并不删除 **working-tree** 中文件,
    这个命令一般是用来移除那些被误引入版本控制中的文件， 比如我们误将*.DS_Store* 文件加到了 git 仓库中，
    现在我们要把这个文件过滤掉， 可通过以下两步：

        1. 在 *.gitignore* 文件中加入 *.DS_Store*, 将这个文件过滤掉;
        2. `git rm --cached .DS_Store` 将这个文件从 index 中删除;

    3. `git clean` 从 **working-tree** 中删除，一般用来删除未跟踪的文件，这个过程是不可回退的，
    既然是未跟踪文件，所以没有索引值，会将当前工作目录下的文件一并删除，所以万分注意；
    4. `rm` 相当于就是从 **working-tree** 中删除文件，此过程也是可回退的;

2. `git commit -am "******"`

    `-a` 参数是针对已跟踪文件的提交，对于未跟踪的文件需要先 `git add file-name` 后才能 `commit` 上去

3. `git checkout -- <filename>`

    这个命令是用当前分支 `HEAD` 中的内容替换掉当前工作目录中的内容，stage 状态和未跟踪文件不受影响;
    要将 stage 状态文件恢复到 unstaged 状态: `git reset HEAD -- <filename>`

4. `git fetch origingit reset --hard origin/master`

    如果想要丢弃所有本地的改动和提交，可以到服务器上获取最新的版本并将本地分支指向它
5. `git blame file`

    查看一个文件的更改记录 等价于 `git log file`

6. `git branch -D`

    强制删除一个分支和这个分支上未被合并的 commits，即使这个分支上的提交没有被 merge 上去。
    删除一个分支后，这个分支上已提交的 commits 依然存在， 未提交的 commits 将丢失; 所以使用 `git branch -d`
    删除一个分支的前提是该分支上的提交都已经被 merge 到其它分支上了。

7. `git diff`

    这个命令比较的是 working-tree 与 stage 状态之间的差异；stage 状态对应 *.git/index* 文件；
    `git diff --staged` 比较的是已经暂存(stage)起来的文件和上次提交的版本之间的差异.

8. `git push --set-upstream origin master`

    推送的同时将本分支设为 pull 时的上流分支, 这个命令一般用在将本地某一分支推送到仓库上，
    同时将该分支设为远程跟踪分支

9. 克隆指定的远程分支, 如果你希望只克隆远程仓库的一个指定分支，而不是整个仓库分支:

        git init
        git remote add --track BRANCH_NAME --feature origin REMOTE_REPO_URL_PATH
        git checkout BRANCH_NAME

10. `git push [remote_name] [local_branch]:[remote_branch]`

    如果省略 `local_branch`，那么就相当于删除对应的远程分支 `remote_branch`

11. 使用 `rebase` 推送而非 `merge`

    如果您正在团队中工作并且整个团队都在同一条 **branch** 上面工作，那么您就得经常地进行 `fetch, merge, pull`。
    Git中，分支的合并以所提交的 `merge` 来记录，以此表明一条特性分支何时与主分支合并。
    但是在多团队成员共同工作于一条 branch 的情形中，常规的 `merge` 会导致 log 中出现多条消息，从而产生混淆。
    因此您可以在 `pull` 的时候使用 `rebase`，以此来减少无用的 `merge` 消息，从而保持历史记录的清晰;
    对于已经 `push` 上去的 commits 是不建议使用 rebase:

        git pull --rebase

    您也可以将某条 branch 配置为总是使用 rebase 推送：

        git config branch.BRANCH_NAME_HERE.rebase true

12. `git stash`

    1. 当我们当前分支工作时想跳到另一个分支去，但是又不想提交当前分支，可以用 `git stash`（或者 `git stash save`）
    把未提交的本地修改先保存入栈，也就是将工作目录中未提交的记录入栈; 当跳回这个分支时再用 `git stash apply/pop`
    将修改回退出来；
    2. 用 `git stash list` 可以查看到栈的记录，如果我们想恢复刚才的改动，可以使用 `git stash apply`
    或者 `git stash pop`， 虽然都能恢复到之前的改动，但是这两者还是有区别的：

        1. git stash apply 只是把 stash 栈里面记录的内容运用到当前的版本上，这时候栈里面的记录还保存着。
        2. git stash pop 也是把 stash 栈里面记录的内容运用到当前的版本上，但是它会把出栈的记录删掉，
        也就是说如果之前栈里面有3个记录，当执行 `git stash pop` 之后，栈里的记录变成2个，这就是 pop （出栈）的特点

    3. `git stash` 只能保存本地未提交的已跟踪文件清单，若是本地未跟踪的文件不能 stash；
    若在 `stash` 后对已经 `stash` 的文件作修改并提交后，再将 `stash` 的文件 `apply/pop` 出来的时候将引起冲突。

13. 使用 git 打补丁:

    1. 使用 `git diff`

        1. `git diff > 1.patch` 生成一个补丁包 *1.patch*；
        2. `git apply 1.patch`

        **Hint: `git apply` 要么将所有补丁全部打上，或者全部放弃。**

    2. 使用 `git format-patch`

        1. git format-patch 生成一个补丁包；
        2. git am 1.patch 这种方法生成的补丁包中含有补丁开发者的名字，应用补丁时名字会被记录到版本库中。

##子模块
当 `clone` 一个带 **submodule** 的仓库时，`clone` 后需要先做 `git submodule init` 后再做
`git submodule update` 者直接合并成一条指令:

    git submodule update —init

更新每个子模块:

    git submodule foreach git pull origin master

> No submodule mapping found 错误处理：

    $git submodule init
    No submodule mapping found in .gitmodules for path 'bundle/vim-powerline'
    $git rm --cached bundle/vim-powerline

如果要完全删除一个子模块，需要分步执行几下几步：

    1. *.gitmodules* 中删除相应的子模块, 这步执行之后可以先将当前状态 stage 一下: `git add .gitmodules`；
    2. *.git/config* 中删除相应的子模块；
    3. `git rm --cached path_to_submodule` 从 index 中删除相应的子模块；
    4. `rm -rf .git/modules/path_to_submodule`
    5. `git commit -m` 将 *.gitmodules*
    6. `rm -rf path_to_submodule` 完全删除未跟踪的子模块

从当前项目中递归删除一个文件 Remove existing files from the repository:

    find . -name .DS_Store -print0 | xargs -0 git rm --ignore-unmatch

