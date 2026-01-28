## 1. init

`git init`：当新创建一个工程之后，要在其中创建一个.git文件夹，就使用这个命令。
## 2. remote -v

`git remote -v`：确认远程仓库地址。
## 3. branch

`git branch`：确认仓库内分支。
## 4. add

`git add [File_name]`：选中要提交的文件。

`git add .`：选中除了`.gitignore`文件中提到的文件外所有的文件/文件夹。

## 5.  status

`git status`：确认目前在本地修改后哪些文件被选中，哪些未被选中。

## 6. commit -m "[summary]"

`git commit -m "[summary]"`：将选中的文件提交，并给它写一个概要（summary）。

## 7. switch [branch name]

`git switch [branch name]`：切换到【branch name】分支。

## 8. branch [new branch name]

`git branch [new branch name]`：在本地仓库中创建一个新的分支。

## 9. push --set-upstream origin [branch name]

`git push --set-upstream oringin [branch name]`：为本地分支和远程分支建立关联关系。

## 10. rm -r --chached

`git rm -r --cached *`：将本地.git文件夹中之前抓取的文件全部删除

## 11. push & pull

`git push`：将本地.git文件夹中抓取的文件全部上传到远程。

`git pull`：将远程的目标库的目标分支中的文件更新到本地。

## 12. notepad .gitignore

`notepad .gitignore` ：创建一个.gitignore 文件，用记事本打开，在其中写入不想上传到远程的文件/文件夹。

