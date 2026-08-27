# Git分支发布场景面试版总结
## 场景
环境流水线：fat → uat → pre → pro
- 自己特性分支：`feat998`，基于旧基线开发中
- 其他同事分支：`feat110` 已经合并，成功上线pro生产环境
- 问题：我的feat998上线pro前，是否需要合并master？

### 核心结论
**必须先同步master基线，但不是直接把feat998合并进master。**
> master/main 是生产环境代码基线，feat110上线后已经合入master；我的feat998创建时间早于feat110提交，本地分支缺少feat110变更。
如果直接把旧feat998部署pro，会覆盖丢失feat110线上代码，造成线上bug。

### 标准正确流程（面试口述）
1. 更新本地master，拉取远程最新代码（包含已上线feat110）
2. 切回自己的特性分支`feat998`，**将master合并到feat998**，在特性分支解决代码冲突
3. 推送远程分支，依次部署fat/uat/pre环境，做完整回归：既要测自己业务，也要验证原有feat110功能正常
4. **所有环境测试全部通过之后**，再将feat998合并到master，由master发布pro生产环境

> 关键点：冲突在特性分支解决，保护master基线永远是可发布状态；禁止未测试代码直接合入master。

### merge vs rebase 选型
- `git merge master`：团队协作分支首选，不修改提交历史，冲突生成一次merge commit；多人共用分支只能用merge。
- `git rebase master`：单人私有开发分支可用，提交历史干净；**分支已经推送到远程多人协作严禁rebase**，会打乱他人提交记录。

### 高频踩坑点（风险清单）
|错误做法|风险后果|
|---|---|
|旧特性分支不同步master直接部署pro|丢失其他已上线功能，线上故障|
|未测试直接把feat分支合并到master|master被污染，基线不可用，回滚成本巨大|
|远程共享分支执行rebase|提交历史改写，团队其他人拉代码大量冲突|
|合并master之后不做回归测试|冲突解决不完整，隐藏逻辑bug流到pre/pro|

### 一句话面试简答
> 我需要先把最新master合并到我的feat998分支，拿到线上已经发布的feat110代码，解决冲突后在各个测试环境完整验证，确认功能没问题，最后才把feat998合并进master发布生产，不能直接拿旧分支上线。

---

## 可直接复制的Git操作命令速查
```bash
#1. 更新本地master到远程最新
git checkout master
git pull origin master

#2. 切回自己分支，合并master
git checkout feat998
git merge master
# 手动解决冲突 → add、commit

#3. 推送远程
git push origin feat998

#4. fat/uat/pre全量测试完毕，再合并到master上线
git checkout master
git merge feat998
git push origin master
