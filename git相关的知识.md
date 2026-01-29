fetch是将远程的仓库的最新提交拉取下来，**注意，只是更新指针，更新本地的一个main指针。** fetch做的是更新指针和.git下面的一个对象库（.git/object)。。。将远端的东西下载到本地。。。
	只有使用merge才会更新到本地工作目录，使用fetch只会将指针和相对应的远端更改进行拉取下来。  当前的工作目录指针还是指向原先的HEAD。。。在运行merge会或者reset等命令才会将其进行强制指向最新的一个HEAD。


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


