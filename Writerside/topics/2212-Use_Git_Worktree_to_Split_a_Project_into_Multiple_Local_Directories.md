# 使用 Git worktree 将同一个项目分裂成多个本地目录

想要在同一个项目中同时使用多个分支，但是又不想频繁切换分支，可以使用 Git worktree 将同一个项目分裂成多个本地目录。

## 创建一个新的工作树

```bash
git worktree add ../new-dir some-existing-branch
git worktree add [path] [branch] 
```

## 删除一个工作树

```bash
git worktree remove ../new-dir
```

或者：

```bash
rm -rf ../new-dir
git worktree prune
```