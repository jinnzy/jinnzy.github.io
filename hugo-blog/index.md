# 使用hugo来搭建个人博客

# hugo初体验

## 安装hugo

[下载链接](https://github.com/gohugoio/hugo/releases)

我这里选择的最新版本`v0.80.0`，选择操作系统对应的版本即可，我这里是选的win10 64位的。

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210101204110.png)

下载后将hugo.exe加入到环境变量中，我是习惯创建一个bin目录，将可执行文件都放到这里。

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210101214825.png)

> ps 下载的时候发现链接竟然打不开了，可以通过添加hosts解决

```Bash
# github
199.232.96.133 avatars0.githubusercontent.com
199.232.96.133 avatars1.githubusercontent.com
199.232.96.133 avatars2.githubusercontent.com
199.232.96.133 avatars3.githubusercontent.com
199.232.96.133 avatars4.githubusercontent.com
199.232.96.133 avatars5.githubusercontent.com
199.232.96.133 avatars6.githubusercontent.com
199.232.96.133 avatars7.githubusercontent.com
199.232.96.133 avatars8.githubusercontent.com
199.232.96.133  user-images.githubusercontent.com
```





## 创建一个hugo项目

```Bash
$ hugo.exe new site jinnzy.github.io.source

```

## 安装主题

这里有两种方法我采用的第一种方法，比较方便，有什么问题可以直接修改。

第一种：下载最新的压缩包或直接clone到themes目录

```Bash
$ cd jinnzy.github.io.source

$ git clone https://github.com/dillonzq/LoveIt.git themes/LoveIt

```

第二种：使用子模块来安装

```Bash
$ cd jinnzy.github.io.source
$ git init
$ git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

## 修改配置

位置：`./config.toml`

可以参考[官方文档中3.1的示例配置](https://hugoloveit.com/zh-cn/theme-documentation-basics/)，由于太大这里就不贴出来了

在这个基础上我额外修改了以下几个配置，其余的就是一些网站标题描述等的就不多说了。

```Bash
# 是否使用 git 信息
enableGitInfo = true

[params] 
  gitRepo = "jinnzy.github.io.source"

[markup]
   # 目录设置
  [markup.tableOfContents]
    # 从1个#开始算标题，默认是两个#开始算标题
    startLevel = 1
    endLevel = 6
```

## 修改文章默认的模板

位置：`./archetypes/default.md`

```YAML
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: false
toc: true
tags: 
- ""
categories:
- ""
---
```

加入了目录标签和分类等。

## 创建第一篇文章

```Bash
$ hugo new posts/hugo-blog-1.md
```

启动进行测试，会报错

```Bash
$ hugo server -D
···
Error: Error building site: failed to render pages: render of "page" failed: execute of template failed: template: posts/single.html:92:124: executing "content" at <partial "function/content.html">: error calling partial: "/Users/tc/Documents/workspace_2020/blog/themes/loveIt/layouts/partials/function/content.html:15:15": execute of template failed: template: partials/function/content.html:15:15: executing "partials/function/content.html" at <partial "function/checkbox.html" $content>: error calling partial: partial that returns a value needs a non-zero argument. 
```

> -D不加也可以，因为前面已经设置 `draft: false` 了。

这里报错是一个bug，已经有人[pr](https://github.com/dillonzq/LoveIt/issues/518)但是并未合并进来，所以要自己手动修改一下。

修改文件：`themes/loveIt/layouts/partials/function/content.html
`

修改后如下

```Go
{{- $content := .Content -}}

{{- if ne "" $content -}}

{{- if .Ruby -}}
    {{- $content = partial "function/ruby.html" $content -}}
{{- end -}}

{{- if .Fraction -}}
    {{- $content = partial "function/fraction.html" $content -}}
{{- end -}}

{{- if .Fontawesome -}}
    {{- $content = partial "function/fontawesome.html" $content -}}
{{- end -}}

{{- $content = partial "function/checkbox.html" $content -}}

{{- $content = partial "function/escape.html" $content -}}

{{- end -}}

{{- return $content -}}

```

随后再次启动，进入[`http://localhost:1313`](http://localhost:1313/) 就可以看到博客首页了。

# 利用github pages部署blog

前置条件：需要安装`git`

## 创建github仓库

登录github创建库

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102120518.png)







![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102120607.png)

我这里是创建了两个库

- `jinnzy.github.io.source` 选择的是私有库(Private)存放hugo源文件，利用github actions来编译生成静态文件推送到`jinnzy.github.io`库中

- `jinnzy.github.io` 选择的是公共库(Public)存放hugo编译后的静态页面，主要是通过github pages功能访问这些静态文件。

## 配置github ssh key

生成ssk key

```Go
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
53:0e:0b:0f:c0:b8:d3:4f:e1:8a:a8:5d:8c:8f:98:cf jinzhy@vlnx107005.firstshare.cn
The key's randomart image is:
+--[ RSA 2048]----+
|   o.            |
|  . ...          |
|   o .o.. .      |
|  o . o+ =       |
| . = +  S .      |
|. o + .  .       |
|.+ +             |
|+.o .            |
| .E              |
+-----------------+

```

- `/root/.ssh/id_rsa` 私钥

- `/root/.ssh/id_rsa.pub` 公钥

打开`jinnzy.github.io.source` 库添加`Secrets` ，名称为`ACTIONS_DEPLOY_KEY`，将`/root/.ssh/id_rsa`的内容复制进去

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102122947.png)

打开`jinnzy.github.io` 库添加`Deploy keys` ，名称随便都行，将`/root/.ssh/id_rsa.pub`的内容复制进去。

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102123413.png)

## 配置github actions

打开`jinnzy.github.io.source`，创建Actions

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102130308.png)

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102130526.png)

全部内容如下：

```YAML
name: Deploy Hugo Site to Github Pages on Main Branch

on:
  push:
    branches:
      - main

jobs:
  build-deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.80.0'
          extended: true # 使用扩展版

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }} # 这里的 ACTIONS_DEPLOY_KEY 则是上面设置 Private Key的变量名
          external_repository: jinnzy/jinnzy.github.io # Pages 远程仓库 
          publish_dir: ./public
          keep_files: false # remove existing files
          publish_branch: main  # deploying branch
          commit_message: ${{ github.event.head_commit.message }}

```

## 上传hugo项目传到github

当前所在目录：`./jinzhy.github.io.source`

```Go
$ git init
$ git commit -m "first commit"
$ git branch -M main
$ git remote add origin https://github.com/jinnzy/jinzhy.github.io.source.git
$ git pull # 先把之前提交的的内容拉取下来
$ git push -u origin main
```

进入`Actions` 可以看到上传成功了

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102131715.png)

打开`jinzhy.github.io` 项目

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102131949.png)

下拉找到`GitHub Pages` 可以看到发布地址，这时访问`https://jinnzy.github.io` 就可以看到博客了。

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102132734.png)

# Console报错找不到`/site.webmanifest`

> 引用自[lewky.cn](https://lewky.cn/tags/hugo/)博客中的内容。

该文件和`Progressive web applications (PWA)`有关，通过添加PWA到Hugo站点，可以实现离线访问的功能，也就是说断网状态下依然可以访问到你之前访问过的网页，换言之就是通过PWA来将访问过的网页资源缓存到了本地，所以断网下仍然可以继续访问网站。当然，恢复网络时会自动更新最新的页面资源。

但是目前该功能还不够完善，可能存在着安全性的问题，并且实现过程也比较繁杂，最终还是决定把这个引用给去掉，做法如下：

- 把博客主题目录下的`\themes\LoveIt\layouts\partials\head\link.html`拷贝到根目录下的`\layouts\partials\head\link.html`

- 打开拷贝后的`link.html`，把`<link rel="manifest" href="/site.webmanifest">`删掉或者注释掉：

```YAML
{{- /*    <link rel="manifest" href="/site.webmanifest"> */ -}}

```

# 添加友链

> 注意：参考了Reference中的几篇文章添加友链，发现`css`样式总是不生效，查了一下午才在这个[pr](https://github.com/dillonzq/LoveIt/pull/546)中发现`loveit` 主题某些功能需要把`scss` 转换为`css` 所以要选择扩展版下载，所以又重新下载了`hugo_extended_0.80.0_Windows-64bit` 这个版本😭。

主要使用[kkkgo/hugo-friendlinks](https://github.com/kkkgo/hugo-friendlinks)项目中的代码。

## 添加友链样式

位置：`jinnzh.github.io.source/assets/css/_custom.scss` 

```Sass (scss) 
// friendslink
.myfriends {
  text-align: center;
  background-color: #fff;
  opacity: 0.9;
}

.myfriends a {
  color: black;
}

.myfriends p {
  display: none;
}

.friendurl {
  text-decoration: none !important;
  color: black;
}

.myfriend {
  width: 56px !important;
  height: 56px !important;
  border-radius: 50%;
  border: 1px solid #ddd;
  padding: 2px;
  box-shadow: 1px 1px 1px rgba(0,0,0, .15);
  margin-top: -12px !important;
  margin-left: 10px !important;
  background-color: #fff;
}

.frienddiv {
   position: relative;
   left: 0; right: 0;
   width: 100%;
   height: 70px;
   line-height: 1.38;
   margin-top: -5px;
   margin-bottom: -5px;
   border-radius: 5px;
   background: rgba(255, 255, 255, .2);
   box-shadow: 4px 4px 2px 1px rgba(0, 0, 255, .2);
   overflow: hidden;
}

.frienddiv:hover {
  background: rgba(87, 142, 224, 0.15);
}

.frienddiv:hover .frienddivleft img {
  transition: .9s!important;
  -webkit-transition: .9s!important;
  -moz-transition: .9s!important;
  -o-transition: .9s!important;
  -ms-transition: .9s!important;
  transform: rotate(360deg)!important;
  -webkit-transform: rotate(360deg)!important;
  -moz-transform: rotate(360deg)!important;
  -o-transform: rotate(360deg)!important;
  -ms-transform: rotate(360deg)!important;
}

.frienddivleft {
  width: 92px;
  float: left;
}

.frienddivleft {
    margin-top: 15px;
    margin-right: 2px;
}

.frienddivright {
  margin-top: 15px;
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}
```

## 创建shortcodes友链文件

位置：`jinnzh.github.io.source/layouts/shortcodes/friend.html`

```Sass (scss) 
{{ if .IsNamedParams }}
<p><a target="_blank" href={{ .Get "url" }} title={{ .Get "name" }} class="friendurl">
    <div class="frienddiv">
      <div class="frienddivleft">
  <img class="myfriend" src={{ .Get "logo" }} />
      </div>
      <div class="frienddivright">
  {{ .Get "name" }}<br />{{ .Get "word" }}
      </div>
    </div>
  </a>
</p>
{{ end }}
```

## 创建friend md文件

位置：`jinnzh.github.io.source/content/friends.md`

```Sass (scss) 
---
hiddenFromSearch: true
---
# 友链
{{</* friend name="Dillon" url="https://github.com/dillonzq/" logo="https://avatars0.githubusercontent.com/u/30786232?s=460&u=5fc878f67c869ce6628cf65121b8d73e1733f941&v=4" word="LoveIt主题作者" */>}}
```

这里的friend是引用上个步骤中创建的`friend shortcodes`

## 菜单中添加友链

位置：`jinnzh.github.io.source/config.toml`

```Sass (scss) 
[menu]
  [[menu.main]]
    identifier = "friends"
    pre = ""
    post = ""
    name = "友链"
    url = "/friends/"
    title = ""
    weight = 4 
```

- weight按菜单顺序往下排即可。

最终效果

![](https://jinnzy.oss-cn-beijing.aliyuncs.com/img/20210102223525.png)

# Reference

[https://lewky.cn/tags/hugo/](https://lewky.cn/tags/hugo/)

[https://hugoloveit.com/zh-cn/categories/](https://hugoloveit.com/zh-cn/categories/)

[kkkgo/hugo-friendlinks友链](https://github.com/kkkgo/hugo-friendlinks)

[loveit自定义样式](https://hugoloveit.com/zh-cn/theme-documentation-basics/#33-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%A0%B7%E5%BC%8F)
