## 命令

### 基本

- 第一台设备的初次设置

```bash
1. 给远程仓库起别名
	git remote add origin 远程仓库地址
2. 向远程推送代码
	git push -u origin 分支
```

- 新设备的初次设置

```bash
1. 克隆远程仓库代码
	git clone 远程仓库地址 (内部已实现git remote add origin 远程仓库地址)
2. 切换分支
	git checkout 分支
```

- 新建dev分支并进行开发

```bash
1. 切换到dev分支进行开发
	git checkout dev 
2. 把main分支合并到dev [仅一次]
	git merge main
3. 修改代码
4. 提交代码
	git add . 
	git commit -m 'xx'
	git push origin dev 
```

- 每次更换设备时，如果对应分支的云端代码已经经过了迭代，则需要进行拉取

```bash
1. 切换到dev分支进行开发
	git checkout dev 
2. 拉代码
	git pull origin dev 
3. 继续开发

4. 提交代码
	git add . 
	git commit -m 'xx'
	git push origin dev 
```

- 开发完成后进行合并与推送

```bash
1. 将dev分支合并到master，进行上线
	git checkout master
	git merge dev 
	git push origin master
2. 把dev分支也推送到远程
	git checkout dev
	git merge master 
	git push origin dev 
```

```bash
git pull origin dev
等价于
git fetch origin dev
git merge origin/dev 
```

### rebase（变基）

- rebase可以保持提交记录简洁，不分叉

```bash
git rebase 分支
```

将版本记录用图形展示，可以更好的理解rebase

```bash
git log --graph --pretty=format:"%h %s"
```

## 冲突

- 发生在合并时，需要手动解决，或使用辅助软件（如：beyond compare）

### beyond compare使用步骤

1. 安装beyond compare 

2. 在git中配置

   ```bash
   git config --local merge.tool bc3
   git config --local mergetool.path '/usr/local/bin/bcomp'
   git config --local mergetool.keepBackup false
   ```

3. 应用beyond compare 解决冲突

   ```bash
   git mergetool 
   ```

## 多人协作

- 协同开发时，需要所有成员都可以对同一个项目进行操作，需要邀请成员并赋予权限，否则无法开发。github支持两种创建项目的方式（供多人协同开发）

1. **合作者**，将用户添加到仓库合作者中之后，该用户就可以向当前仓库提交代码。
2. **组织**，将成员邀请进入组织，组织下可以创建多个仓库，组织成员可以向组织下仓库提交代码。

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806093802227.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806095639748.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806121845153.png)

### Tag标签管理

- 为了能清晰的管理版本，在公司不会直接使用commit来做版本，会基于Tag来实现：v1.0 、 v1.2 、v2.0 版本。

```bash
git tag -a v1.0 -m '版本介绍'        创建本地创建Tag信息
git tag -d v1.0                     删除Tag
git push origin  --tags             将本地tag信息推送到远程仓库
git pull origin  --tags             更新本地tag版本信息

git checkout v.10                   切换tag
git clone -b v0.1 地址               指定tag下载代码
```

### code review

1. 配置，代码review之后才能合并到dev分支

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806122224316.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806122307569.png)

2. 完成开发后申请code review（发起一个pull request），申请时可以进行一定的设置，如：指定给谁来做

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806122501088.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806122733005.png)

3. 被指定者（可以是组长或者其他成员）做code review（判断是否同意pull request）

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190806123428479.png)

## 向其他项目提供代码

1. fork源代码，将别人源代码拷贝到我自己的远程仓库
2. 在自己仓库进行修改代码
3. 给源代码的作者提交申请（pull request）

## 其他

- issues，文档以及任务管理，其他人向该项目提问题或建议的渠道，开发者可以进行回答或关闭，也可以进行指派
- wiki，项目文档，用于描述项目的基本概况和提供相关知识

## 一图流

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/image-20190803104819552.png)