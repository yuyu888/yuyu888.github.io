---
layout: mypost
title: 解决git revert后再次merge代码丢失问题
categories: [GIT]
---

## 问题场景

有时我们把分支代码合并到master后，由于各种各样的原因没有能完成上线，这时就需要通过 git revert 把master的代码回归回去， 但是当我们再次上线时， 需要把之前的分支再次merge到master， 这时会发现， 只能把上次合并后新提交的代码合并到master，上次合并前的代码丢了

## 手工解决

1、从master上创建个新分支B;   
2、进入本项目目录， 把本分支A代码cp一份，创建新的目录命名为 A_bak, 删除 .git 文件；   
3、再把git checkout B 切换分支到B分支  
4、然后把 A_bak目录下的文件copy 复制到当前项目目录
5、commit 提交， 然后把 B 分支提交合并到master

## git 命令法

````shell
# 切换到master分支
git checkout master
git pull
# 基于master拉出一个分支 revert_tmp
git checkout -b revert_tmp

# 将之前git revert那次commit再次revert（commit号从git log可以查到）
git revert acd414e1cd42315ce93a9730db961155be140013
git checkout feature-member
git merge revert_tmp
git commit -m "revert"
git push
````