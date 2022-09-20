### 写在前面
目前IDE上操作git是更为便捷有效的手段，此处作为基础了解，业务上还是推荐直接使用IDE。

### 创建本地仓库

- 在项目目录位置执行git初始化操作
  - ```shell
    git init //前往指定项目目录内执行
    ```
- 添加项目描述文件（可选）
  - ```shell
    git add README.md //为项目添加描述文件需要自己创建
    ```
- git.add只是将文件从工作区提交到版本暂存区，需要使用commit提交到仓库区
  - ```shell
    git commit -m "project init"
    ```
- 关联远程仓库与本地仓库
  - ```shell
    git remote add origin https://github.com/Kolloge/TrustInCS.git
    ```
- 将本地仓库区内容提交到远程仓库
    - ```shell
      git push -u origin "master"
      ```

### 拉取远程仓库内容

- 克隆远程仓库内容，拉取所有分支
    - ```shell
      git clone https://github.com/Kolloge/TrustInCS.git
      ```
- 克隆指定分支
    - ```shell
      git clone -b test https://github.com/Kolloge/TrustInCS.git
      ```
    
### 查看状态

- 查看工作区状态，可见分支等
    - ```shell
      git status
      ```
- 查看远程仓库信息
    - ```shell
      git remote -v
      ```

### 分支操作

- 切换到指定分支
    - ```shell
      git checkout develop
      ```
- 创建并切换到指定分支
    - ```shell
      git checkout -b develop
      ```
- 查看本地分支
    - ```shell
      git branch
      ```
- 查看本地和远程分支
    - ```shell
      git branch -a
      ```
- 删除本地分支
    - ```shell
      git branch -d develop
      ```
- 创建远程分支
  - ```shell
      git push origin develop
      ```
- 删除远程分支
    - ```shell
      git push origin --delete develop
      ```