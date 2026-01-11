# Hugo + Github Pages 构建书站

## 准备仓库

### 创建仓库
注意：
- 仓库名：repo-name.github.io
- 权限：public
- 拉取到本地

### Github Pages
- Build and deployment ：Source 更改为 Github Actions

### Github Actions
- New Workflow：搜索hugo，点击Configure进行配置。直接点击Commit
- 现象：在Code仓库下出现.github的文件夹

## 准备Hugo

### 下载Hugo
- 官网下载
- 选择 expend 版本
- 安装后加入path变量

### 初始化网站
- 在本地的 repo-name.github.io 文件夹下执行：
```shell
hugo new site blog
```
- 将blog目录中的内容全部移动到 repo-name.github.io

### 网站主题
- 添加自己喜欢的主题，eg：
```shell
git submodule add https://github.com/hugo-fixit/FixIt.git themes/FixIt
```
- 应用主题，eg：
  编辑 hugo.toml 配置文件，在最后添加一行 theme = "FixIt" 

### 配置网站
- hugo.toml
可以配置网站布局、内容权重等
- hugo.toml的内容可以从选取的主题项目中去找
- 配置baseURL，eg：```shell baseURL = 'https://repo-name.github.io/'``` 最后的 / 别丢了

#### 注意
- 目录层级：目录下初始化_index.md + 设置toml的权重 ```shell hugo new content.zh/docs/go/_index.md ```
- 文章语言：toml支持多语言下，文章各自管理 

### 构建网站
- 直接git push到远程仓库后，进行自动构建。
- 可以访问 https://repo-name.github.io
#### 注意
- 每次构建后要看构建结果，如果构建失败，查看可能构建失败原因。

### 更新tips
- .gitignore 可以排除项：(基本)
  public/
  .hugo_build.lock

# 博客维护

## 内容

### 新增目录

1. /content.zh/docs 下增加，比如：云原生
2. 添加_index目录，说明是可展开目录，比如：
```shell
hugo new content.zh/docs/go/_index.md
```
3. 本地，直接 hugo server / hugo server -D(包括草稿)

### 新增内容
1. //content.zh/docs/* 下增加，比如：/docs/go/go.md || /docs/go.md
```shell
hugo new content.zh/docs/go/go.md
```
2. 注意修改 draft (draft = true为草稿)，否则为正式发布文章

### 更新远程
本地更新后，直接推到远程，远程自动构建。

## 样式
暂且这样...

