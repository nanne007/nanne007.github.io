---
title: Git 实用技巧
date: 2013-09-09T21:15:06+08:00
description: 记录 git 使用过程中积累的一些技巧
tags:
- git
---


### 使用 git 比较非 git 项目的文件 ###


`git diff --color-word --no-index file1 file2`

主要是加 `--no-index` 参数。

超级方便，不需要再借助其他命令了。


### 暂存部分修改 ###

使用 git 的时候，偶尔会需要暂存修改的一部分。
stack overflow上有一个相关的[问题](http://stackoverflow.com/questions/1085162/how-can-i-commit-only-part-of-a-file-in-git)。

这是community wiki的解答。

> You can do `git add -p filename.x`, and it'll ask you what you want to stage. You can then:
>
> hit `s` to split whatever change into smaller hunks. This only works if there is at least one unchanged line in the "middle" of the hunk, which is where the hunk will be split.
>
> then hit either:
> - `y` to stage that hunk, or
> - `n` to not stage that hunk, or
> - `e` to manually edit the hunk (useful when git can't split it automatically)
> - and `d` to exit or go to the next file.
> - use `?` to get the whole list of available options.
>
> If the file is not in the repository yet, do first `git add -N filename.x`. Afterwards you can go on with `git add -p filename.x`.

主要是使用 `git add -p filename`命令。git manual 给出了详细解释：

```
       -p, --patch
           Interactively choose hunks of patch between the index and the work
           tree and add them to the index. This gives the user a chance to
           review the difference before adding modified contents to the index.

           This effectively runs add --interactive, but bypasses the initial
           command menu and directly jumps to the patch subcommand. See
           "Interactive mode" for details.
```

wiki最后还提到了 `git add -N filename.x`。`-N`的解释是：

```
       -N, --intent-to-add
           Record only the fact that the path will be added later. An entry
           for the path is placed in the index with no content. This is useful
           for, among other things, showing the unstaged content of such files
           with git diff and committing them with git commit -a.
```


### 撤销上一次提交 ###

> [how to undo the last git commit](http://stackoverflow.com/questions/927358/how-to-undo-the-last-git-commit)

使用命令 `git reset [|--hard|--soft] HEAD~1`, 注意参数的区别：`--hard`, `--soft`。

### git apply -p1 ###

今天尝试了下使用 git apply 命令，报错 `No such file or directory`。
了解了一下，是 patch 里面的路径不对。
因为patch 是用 `diff` 命令直接生成的，文件路径和 git 路径不一致。
在 apply 的时候需要使用 `-p` 参数，移除文件路径前面的部分。

```
       -p<n>
           Remove <n> leading path components (separated by slashes) from traditional diff paths. E.g., with -p2, a patch against a/dir/file will be applied
           directly to file. The default is 1.
```

