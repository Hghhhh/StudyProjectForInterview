#git学习

##创建版本库

1.初始化仓库：`git init`

2.添加文件到Git仓库:

- 使用命令`git add <file>`，注意，可反复多次使用，添加多个文件；

- 使用命令`git commit -m <message>`，完成。

##时光机穿梭

3.`git status`命令可以查看仓库当前的状态

4.`git diff <file>`可以查看当前版本对应上次版本修改的内容

### 版本回退

5.`git log`命令可以显示从最近到最远的提交日志，`git log --pretty=oneline`返回精简的信息

```git
$ git log
commit 26c89291540821b23ad4e8c47e48336d5c4ab57d (HEAD -> master)
Author: ***
Date:   Mon Apr 1 12:27:35 2019 +0800
    v2
commit c59f170cdc1decc8215c630d2e0db2d23387d930
Author: ***
Date:   Mon Apr 1 12:22:08 2019 +080
    test
```

一大串类似`c59f170...`的是`commit id`（版本号）,有了它可以进行提交版本的回退

6.`git reset`回退版本，通过移动HEAD指针来改变当前版本

```git
#回退到前一个版本
git reset HEAD^ 
#回退到前前个版本
git reset HEAD^^
#回退前面N的版本
git reset HEAD~N
#让HEAD指针指向某个版本,这里的c59f170是commitid的前面几位
git reset --hard c59f170
```

7.reset版本的时候，这个版本之后的版本通过`git log`会查不到，这时候可以通过`git reflog`来查看之前每一次的命令，从而找到版本号

## 工作区和暂存区

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/git.jpg)

工作区就是我们电脑能看到的目录

暂存区（stage）是我们git add时候存放的地方

Head和分支是我们commit的时候存放的地方

### 撤销修改

8.当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`

9.当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD <file>`，回到8这种情况。

10.已经提交了不合适的修改到版本库时，想要撤销本次提交，参考[版本回退](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013744142037508cf42e51debf49668810645e02887691000)一节，不过前提是没有推送到远程库

### 删除文件

11.删除了工作区中的文件之后，现在你有两个选择：

- 一个是确实要从版本库中删除该文件，那么使用`git rm <file>`，然后commit
- 另一种情况是删错了，需要恢复文件，使用`git checkout --<file>`可以将文件从版本库里面恢复到工作区，但可能会丢失最近一个提交后你修改的内容

## 远程仓库

###连接远程仓库

12.`git remote add origin git@server-name:path/repo-name.git`，其中origin是远程仓库名称，后面是远程仓库的地址

13.关联后使用命令`git push -u origin master`,第一次推送master分支的所有内容。第一个推送master分支时加-u参数，Git不但会把本地的master分支内容推送到远程新的master分支，还会把本地master分支和远程的master分支关联起来，在以后的推送或拉取时就可以简化命令。

### 从远程库克隆

14.`git clone git@github.com:Hghhhh/Hghhhh.git0`

你也许还注意到，GitHub给出的地址不止一个，还可以用`https://github.com/XXX/XXX.git`这样的地址。实际上，Git支持多种协议，默认的`git://`使用ssh，但也可以使用`https`等其他协议。

使用`https`除了速度慢以外，还有个最大的麻烦是每次推送都必须输入口令，但是在某些只开放http端口的公司内部就无法使用`ssh`协议而只能用`https`

## 分支管理

###创建和并分支

15.查看分支：`git branch`

16.创建分支：`git branch <name>`

17.切换分支：`git checkout <name>`

18.创建+切换分支：`git checkout -b <name>`

19.合并某分支到当前分支：`git merge <name>`

20.删除分支：`git branch -d <name>`

### 解决冲突

当我们使用两个分支的时候，各自都有提交，像下面的图片所示，Git无法”快速合并“，只能试图把各自的修改合并起来，但这种合并就可能会有冲突，这时候就需要手动处理冲突。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/gitConflit.png)

```git
$ git merge test
Auto-merging test.txt
CONFLICT (content): Merge conflict in test.txt
Automatic merge failed; fix conflicts and then commit the result.
#这时候通过git status看到冲突的文件
$ git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)

        both modified:   test.txt
    
#查看该文件，可以看到两个分支修改的部分，手动处理一下冲突，重新提交即可    
$ cat test.txt
hello
<<<<<<< HEAD
world &312
=======
world & 123
>>>>>>> test

```

21.用`git log --graph --pretty=oneline --abbrev-commit`命令可以看到分支合并图。

### 分支管理策略

通常，合并分支时，如果可能，Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

22.`git merge --no-ff`，加上`--no-ff`参数就可以用普通模式合并，合并后的历史有分支，能看出来曾经做过合并

23.在实际开发中，我们应该按照几个基本原则进行分支管理：

- 首先，`master`分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

- 那在哪干活呢？干活都在`dev`分支上，也就是说，`dev`分支是不稳定的，到某个时候，比如1.0版本发布时，再把`dev`分支合并到`master`上，在`master`分支发布1.0版本；

- 你和你的小伙伴们每个人都在`dev`分支上干活，每个人都有自己的分支，时不时地往`dev`分支上合并就可以了。

### Bug分支

24.当我们需要切换到其他分支或创建一个另外的分支处理bug，但当前分支的工作区还没有办法提交，因为工作只是进行到一半，这时候可以使用`git stash`；来保存工作现场，然后就可以切换其他分支处理bug了。

25.处理完bug之后，我们切回原来的分支，这时候需要恢复工作区，先通过`git stash list`查看工作现场，然后有两个办法恢复：

- 一是用`git stash apply`恢复，但是恢复后，stash内容并不删除，你需要用`git stash drop`来删除；

- 另一种方式是用`git stash pop`，恢复的同时把stash内容也删了：

```git
$ git stash 
$ git checkout -b test
$ git checkout master
$ git stash list
stash@{0}: WIP on master: 8cb8206 31
$ git stash apply stash@{0}
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
        modified:   test.txt
no changes added to commit (use "git add" and/or "git commit -a")

```

###Feature分支

26.软件开发中，总有无穷无尽的新的功能要不断添加进来。添加一个新功能时，你肯定不希望因为一些实验性质的代码，把主分支搞乱了，所以，每添加一个新功能，最好新建一个feature分支，在上面开发，完成后，合并，最后，删除该feature分支。

27.如果要丢弃一个没有被合并过的分支，可以通过`git branch -D <name>`强行删除。

### 多人协作

28.多人协作的工作模式通常是这样：

1. 首先，可以试图用`git push origin <branch-name>`推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！

如果`git pull`提示`no tracking information`，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to <branch-name> origin/<branch-name>`。

## 忽略特殊文件

29.在Git工作区的根目录下创建一个特殊的`.gitignore`文件，然后把要忽略的文件名填进去，然后commit这个文件，Git就会自动忽略这些文件。

```git
$ git add hello.txt
The following paths are ignored by one of your .gitignore files:
hello.txt
Use -f if you really want to add them.
```

30.如果你确实想添加该文件，可以用`-f`强制添加到Git

31.或者你发现，可能是`.gitignore`写得有问题，需要找出来到底哪个规则写错了，可以用`git check-ignore`命令检查：

```git
$ git check-ignore -v hello.txt
.gitignore:1:hello.txt  hello.txt
```



这是一篇[廖雪峰Git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)的笔记，这个教程写的非常好，简单易懂！