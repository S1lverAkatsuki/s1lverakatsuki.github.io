# Git 原理 & 简单应用

是对视频[看完这个，你终于能理解 Git 了 | LearnThatStack](https://www.bilibili.com/video/BV1va2ZBzEhx/?share_source=copy_web&vd_source=e07ae0eaa55b638cf9dcf06b4067d587) 的总结。

> 2026-01-13 新增一些常见操作场景的记录

## 提交

Git 操作数据的基础单位。
有个指向前一个提交的头指针，提交构成的图是有向无环图，也就是不能互为父提交

## 分支

指向某个提交的哈希值的指针，诸如 main 分支的内容实际上就是当前 main 分支上最新提交的哈希。

## HEAD（头指针）

指向分支的指针。

![alt text](/posts/imgs/faZAZm8S1QygTQcoACcSU7.png)

图里面的 HEAD 指向 main 分支指向的提交 d4e5f6g

使用 `git checkout <分支名/哈希值>` 来移动 HEAD 指针位置。
当直接指向某个哈希值所对应的节点时，这时中间没有分支衔接，称为分离 HEAD 状态。

![alt text](/posts/imgs/f2Fx49LXf9VVqbUPt2aquM.png)

HEAD 直接指向 b2c3d4e，此时新提交无法被持久化，30-90 天后被垃圾回收清理。

## 工作区，暂存区，仓库

工作区是你正在编写的代码，暂存区是准备提交的文件索引所在地，仓库是永久的数据库，里面存放着往期的提交。

`git add` 工作区存入暂存区，`git commit` 暂存区存入仓库。

## 对命令的理解

### `git checkout`

![alt text](/posts/imgs/g61FYRJoi1k1Xx2kdVyM2A.png)

单纯移动头指针，不改变历史记录。

### `git reset`

![alt text](/posts/imgs/w4Ujq51qdBSLAB212CKcGS.png)

移动分支的标签，比分支标签所在的提交晚的提交会准备进垃圾回收。字如其名一般用来撤销。
有三个参数

- `--soft` 移动后撤销的提交（这里的 c2, c3）会进到暂存区中，原有暂存区不变，工作目录也不变。用来合并多个提交到一个里面（c2，c3 合并到一个跟在 c1 后的新提交 c4）
- `--mixed` 默认操作。暂存区重置，更改还在文件当中，只是没有暂存。用于重新提交当前已提交的内容，诸如改改 commit message，或者是拆成多个提交。
- `--hard` **很危险的操作**，重置暂存区，工作目录里的文件也会删了，未提交的就没了。

### `git revert`

向前创建一个全新的提交来抵消之前的更改。

![alt text](/posts/imgs/j9TFWbwFjre8YsGXzffsZA.png)

图中 revert 提交就是创建了一个 c2 提交的相反操作。

### `git rebase`

这是 merge 的结果：

![alt text](/posts/imgs/gtbdKaJcCf614LeRH8YdFf.png)

两个分支合并到一个。
这是 rebase 的结果：

![alt text](/posts/imgs/fwqUtVavuMag56nn7hwxGH.png)

把一个分支的提交再在主分支上做一遍。
注意 B，C 和 B'，C' 只在行为上是一致的，哈希值不同，因为哈希计算要用父节点。
变基后的分支（这里的 feature）会在操作后进垃圾回收。
永远不要在公共分支比如 main 上使用 rebase。因为 rebase 会改变提交的哈希值，git 会认为这是新的提交，尽管内容一致。这样远端同步过去就乱了。

### `git reflog`

搞砸了之后看引用日志，查看 HEAD 的每一次变化和操作。
找到被删除分支的最后一次提交，然后再用 `git branch <分支名> <哈希值>` 重新创建分支。

## 场景记录

直接记解决方案是最直观的办法

### 1

在提交完以后，发现自己还要小小修改一些隶属于刚刚提交的内容（诸如注释这种），这时候不必撤销或是创建新提交。

1. `git add .` 把刚刚修改的内容添加到暂存区
2. `git commit --amend --no-edit` 往刚刚的提交内追加内容，`--no-edit` 代表沿用刚刚的 commit 文本，如果想修改就换成 `-m <文本内容>`

### 2

云端仓库有内容（比如说模板创建的 `LICENCES` 之类的），本地仓库也有内容（写完的代码要推到在线仓库了）。
直接关联合并会出错，因为两个仓库提交历史完全无关。

1. `git pull origin main --allow-unrelated-histories`
2. 处理合并冲突，比如本地也有一个同名的 `LICENCES`
3. 合并完了推上去 `git push -u origin main`
