fetch是将远程的仓库的最新提交拉取下来，**注意，只是更新指针，更新本地的一个main指针。** fetch做的是更新指针和.git下面的一个对象库（.git/object)。。。将远端的东西下载到本地。。。
	只有使用merge才会更新到本地工作目录，使用fetch只会将指针和相对应的远端更改进行拉取下来。  当前的工作目录指针还是指向原先的HEAD。。。在运行merge会或者reset等命令才会将其进行强制指向最新的一个HEAD。


#### 二次学习
1. git fetch origin是将所有的远程分支进行拉取到.git目录，**更新所有的分支指针，让其指向最新的。**（git merge x不会联网）
	1. 如何在合并的时候不使用fetch，就会出现一个问题，使用merge时，其可能同步的不是远端的最新的一个代码。**其同步的是本地的一个最新的分支代码**
	2. 在任何同步的前提下，想要同步远端代码，必须先将远端代码拉取下来更新指针。

2. 其同时也会将所有的本地不存在的分支也给拉取下来，会从远端拉取最新的一个跟踪。**但是不会创建对应的分支，需要自己创建**
	1. 需要使用git switch --track origin/new来进行同步本地不存在的远端分支。
	2. 如果本地不存在会显示是远端的分支。

3. origin/feat和feat两者是不同的。**（一定注意，fetch只是将远端指针进行更新。）**
	1. 当使用git merge origin/feat时，是将远端跟踪的指针进行同步，是将远端的内容合并到当前分支。
	2. 当使用git merge feat时，是将**本地当前的一个指针**进行合并，合并到当前分支。
	3. 两者相同的时候，就是执行fetch后，每个分支都已经进行合并了，或者执行pull命令，而不是fetch命令。
---

### worktree的作用和用法
- worktree和git clone很像。---都避免了频繁的切换分支和避免了暂存stash。
- 可以临时在这里面做实验，不会污染当前的工作区。
一：操作原理
这个是共享了git的obstact对象，**主要是新建了一个工作目录，可以并且的处理不同的任务。** 其git是共享的，比如说git push，也是push到与worktree下来分支的相同。

操作指令：
- git worktree add
- 在删除时必须使用git worktree remove，不要使用人rm -rf，然后在分支上面运行git worktree prune这个命令清理
==总结就是主要就是新建一个工作目录，轻量快捷，与git clone对比不会多次下载git目录==


### git操作的概念
- git存在分支的概念，分支的概念又存在本地分支和远程分支，如果想将本地分支推送到远程分支，需要对upstream远程分支进行关联，如果不关联就必须得使用显示推送，比如
	- git push -u origin main --这个-u就是关联上层upstream，后续可以直接git push就可以。。。
	- 其他的分支也是进行相对应的操作。

### 同一个电脑上多个github账户处理
在push或者操作时，终端会根据url中的用户名进行对齐，将保存的token进行验证。如果对于一个url，上传的时候无法验证，显示无法确认用户或者用户出错。
1. 可以在将remote远端的url进行设置，
	1. git remote set-url origin https://liuhaoran668@github.com/liuhaoran668/ros1.git
	2. 这里 `liuhaoran668@` 是“用于认证的用户名”，后面的`liuhaoran668/ros1.git` 是仓库路径
这样仓库进行准确的进行验证，与token进行对应。


