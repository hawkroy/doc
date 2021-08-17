- 如何回退指定后缀的文件

  `git status -s | cut -f 2 | sed -n -r '/*.$suffix/p' | xarg -i git checkout -- {}`

- merge 不同分支的commitment

  - cherry pick   `git cherry-pick commit-id`, 在没有冲突下可以merge成功
  - git rebase    `git rebase branch/commit-id`, 将rebase分支的信息移入当前分支，并以此为起点
  
- 迁移某个已知库到新库中

  ```bash
  git clone 旧的库 --bare		# 只clone原来库中的提交信息，生成xxx.git目录
  mkdir 新的库
  cd 新的库
  git init --bare
  cd xxx.git
  git push --mirror 新的库
  ```

  