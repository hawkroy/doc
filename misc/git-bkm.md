- 如何回退指定后缀的文件

  `git status -s | cut -f 2 | sed -n -r '/*.$suffix/p' | xarg -i git checkout -- {}`

- merge 不同分支的commitment

  - cherry pick   `git cherry-pick commit-id`, 在没有冲突下可以merge成功
  - git rebase    `git rebase branch/commit-id`, 将rebase分支的信息移入当前分支，并以此为起点