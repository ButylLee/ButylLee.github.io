---
title: "如何在 Github Pages 上使用 Jekyll 主题 Chirpy 搭建个人博客"
description: "博客迁移记录"
date: 2026-04-06 16:00:00 +0800
categories: [经验教程]
tags: [Jekyll, Chirpy, WordPress, 博客]
copyright:
image:
math: false
mermaid: false
pin: false
render_with_liquid: false
---

## 背景

2021 年的时候，我在阿里云上自购了服务器用于搭建个人博客，使用了 WordPress 方案，当时一年的费用还只要一两百，但是 24 年的时候费用越来越高，已经到了五百往上了，遂只能关停。

网站关闭前我用 WordPress 插件 All-in-one-wp-migration 把博客内容备份了下来，所以也顺便写下博客的迁移记录。

## 博客迁移恢复

在本地部署 WordPress，我用的是 Ubuntu 虚拟机，使用 XAMPP（原名 LAMPP）部署，其中集成了所需的 MySQL 和 Apache。具体步骤可以参考[这篇知乎文章](https://zhuanlan.zhihu.com/p/32473851)。

- [XAMPP 官方网站](https://www.apachefriends.org/zh_cn/index.html)

安装好后会打开其管理界面，如果后续需要再次打开的话，需要用管理员权限打开 `/opt/lampp/manager-linux-x64.run`。

在 WordPress 管理页面下载 All-in-one-wp-migration 插件，这个插件备份是免费的，还原如果文件超过一定大小则需要付费，或者自己 Hack 修改容量限制，这个 Hack 其实是作者故意留下的，但据我验证新的版本似乎已经无法手动解除限制了，网上的教程都太老，我用了多种方法依旧失败，最后在 [Github](https://github.com/dev071/all-in-one-wp-migration-unlimited) 上找到了插件的破解版本，才得以恢复。

然后就可以在本地查看还原后的 WordPress 界面了。

## 新博客网站方案

2026 年的时候我看到可以用 Github Pages（github.io）来建立个人博客，主要还是我想要一个比较私人的地方来存放个人文章，于是想起来还有这个免费方案，Github Pages 提供了静态网站的免费托管服务，这正适合个人博客这种应用。

以前 Github Pages 上官方支持直接生成 [Jekyll](https://jekyllrb.com/) 页面，Jekyll 是一个类似于 WordPress 的博客网站框架，但功能要简洁得多，主要支持静态博客网站，博客文章是直接用 Markdown 文件生成的。但现在 Github 页面已经去掉直接生成 Jekyll 的按钮了，因此网上的大部分教程都是失效的。

实际使用 Jekyll 都是直接使用主题模板，大部分的主题都会提供上手文档，可以在[这里](https://github.com/topics/jekyll-theme)挑选主题，我使用了 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题，其上手文档在其[样例网站](https://chirpy.cotes.page/)里。

## Jekyll 配置

### 生成仓库

首先看样例网站里的 [Getting Started](https://chirpy.cotes.page/posts/getting-started/)。我们先从[仓库模板](https://github.com/cotes2020/chirpy-starter)创建自己的页面，点击右上角的 `Use this template` 按钮，选择 `Create a new repository`。仓库的名称要以 `你的用户名.github.io` 作为格式，其他的不用改直接点 `Create Repository`。

然后就生成了相应的仓库，后面的编辑工作我推荐在本地也部署 Jekyll 方便调试预览，使用 VSCode 作为编辑器，每次推送到 Github 的主分支就会进行一次 Github Action 生成网站页面，网址就是 `你的用户名.github.io`。

### 本地部署

首先使用 [RubyInstaller]([Downloads](https://rubyinstaller.org/downloads/)) 安装 Ruby，下载页面中前面带箭头=>的版本。打开后可以更改安装目录，其他的选项不用动，安装后直接点完成，会打开一个安装 MSYS2 的命令行界面，这里三个选项都要安装。

接着打开命令行界面，输入 `gem -v`，若弹出版本号则安装成功。

接着在命令行输入 `gem install jekyll bundler`，等待 Jekyll 包安装完成。输入 `jekyll -v`，若弹出版本号则安装成功。

首先将 Github 博客仓库 clone 到本地，在其目录下右键打开终端（cmd/powershell），或者打开终端切换到对应路径下。输入 `bundle exec jekyll serve` 就可以在本地运行，输出信息中的 `http://127.0.0.1:4000` 就是对应地址，在浏览器中打开即可，对仓库中文件的编辑将会实时增量编译，只要刷新浏览器页面即可，命令行按 `Ctrl+C` 即可退出。

博客的“源代码”是 `_data`、`_layouts`、`_plugins`、`_posts`、`_tabs`、`assets` 以及根目录下的一些文件，在本地部署后会生成 `_site` 文件夹，这个就是编译后的网页文件，可以放心删除。

### 基础配置

首先可以编辑仓库的 `README.md` 文件，该文件初始为 Chirpy 模板仓库自带说明，在生成自己的仓库后可以改成自己的内容，例如添加 License 声明，这里需要注意仓库的主要内容为 Chirpy 主题，其默认 License 是 MIT license，但自己的博客文章及相关附件的创作版权归属于自己，适用于 CC 版权声明，这里我选择的是 `CC BY-NC-SA 4.0` 协议，措辞如下：

```
The contents on xxx.github.io is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) unless otherwise specified.
```

打开目录下的 `_config.yml` 文件，更改博客网站的基础信息配置：

- `lang`：改为 `zh-CN`
- `timezone`：改为 `Asia/Shanghai`，这里的定义是从[这里](https://zones.arilyn.cc)获取的，我也很奇怪为什么没有北京时间
- `title`：网站标题
- `tagline`：网站副标题，可以用 `""` 表示空
- `description`：网站说明，会在搜索引擎中展示
- `url`：你的博客网址
- `twitter`：不需要的可以在最前面加 `#` 注释掉，其子项也需要注释掉
- `social`：根据注释说明更改，这里的 `name` 就是文章界面显示的作者名
- `avatar`：作者头像，后文详述
- `pwa`：渐进式 Web 应用功能，建议关闭，开启后会在本地保存网站缓存，这样下次打开时可以快速加载，甚至没网时也能加载，但问题是国内的环境有时候打开 Github 会很慢甚至打不开，这个时候就不容易分辨网站的时效性，并且启用 PWA 功能后当网页有新内容时，是通过弹出一个框和按钮提醒是否更新来刷新的，如果实际打不开网页，那会造成实际上没有更新但还能正常打开的情况，我个人觉得会引起体验上的不便，因此选择将其关闭。属性 `cache` 也要关闭。

打开 `_data/contact.yml`，这个是博客左下角的联系信息，按顺序显示。其中 Github、邮箱链接在 `_config.yml` 中定义，其他的根据需要更改即可。如果需要增加模板中没有的社交信息模板，例如知乎，只需要单开一行写

```yaml
- type: zhihu
  icon: "fab fa-zhihu"
  url:  "https://www.zhihu.com/people/xxx"
```

其中社交媒体的 logo 是通过 [FontAwesome](https://fontawesome.com/) 获取的，搜索到相应图标界面后复制 HTML 格式中的 class 属性内容即可，其中 `fab` 是 `fa-brands` 的缩写。

打开 `_data/share.yml`，这个文件配置文章末尾的分享按钮。根据需要修改即可。

### 更改头像

参考[此处](https://chirpy.cotes.page/posts/customize-the-favicon/)，首先到 [Real Favicon Generator](https://realfavicongenerator.net/) 上上传你的头像，最好是分辨率为 512x512 的方形头像，上传后直接点击末尾的 Next，然后下载生成的压缩包。

解压后将 `site.webmanifest` 文件删除，然后将剩余的所有文件复制到 `assets/img/favicons/` 下，没有文件夹就创建一个。这样就可以了，页面可能需要重新加载几次才会更新。

### 编辑“关于”页面

打开 `_tabs/about.md` 可以编辑博客的“关于”界面。

### 更改版权信息

博客仓库中的文件其实是不完整的，还有一些公用资源存放在别处，网站生成时会到相应地址获取资源，为了更多样的自定义，我们需要从资源仓库中手动复制一份到自己的仓库中修改，网站生成时会优先使用博客仓库中的配置。例如国际化文件，你会发现自己的博客仓库里面并没有相关的翻译资源文件，其实它们都放在 [Jekyll 主题仓库](https://github.com/cotes2020/jekyll-theme-chirpy/tree/master/_data/locales)下。

从[此处](https://github.com/cotes2020/jekyll-theme-chirpy/tree/master/_data/locales)目录下复制 `zh-CN.yml` 到自己的博客仓库下的 `_data/locales/`。然后你就可以修改文章版权信息了，我使用了 `CC BY-NC-SA 4.0` 协议，更改 `copyright` 子项即可。

如果需要进一步自定义，比如个别文章采用不同的版权声明，那么需要从[此处](https://github.com/cotes2020/jekyll-theme-chirpy/tree/master/_layouts)复制 `post.html` 文件到自己博客仓库下的 `_layouts/`，找到 `<div class="license-wrapper">` 这一行，将

```html
      <div class="license-wrapper">
        {% if site.data.locales[lang].copyright.license.template %}
          {% capture _replacement %}
        <a href="{{ site.data.locales[lang].copyright.license.link }}">
          {{ site.data.locales[lang].copyright.license.name }}
        </a>
        {% endcapture %}

          {{ site.data.locales[lang].copyright.license.template | replace: ':LICENSE_NAME', _replacement }}
        {% endif %}
      </div>
```

改为

```html
      <div class="license-wrapper">
        {% if page.copyright %}
          {{ page.copyright }}
        {% else %}
          {% if site.data.locales[lang].copyright.license.template %}
            {% capture _replacement %}
          <a href="{{ site.data.locales[lang].copyright.license.link }}">
            {{ site.data.locales[lang].copyright.license.name }}
          </a>
          {% endcapture %}

            {{ site.data.locales[lang].copyright.license.template | replace: ':LICENSE_NAME', _replacement }}
          {% endif %}
        {% endif %}
      </div>
```

然后在需要的文章头部信息处（Front Matter）添加 `copyright: "xxx"` 即可。

### 允许包含 http 链接

如果文章中包含有 `http://` 而非 `https://` 链接，Github Actions 在构建页面时会以不安全为由拒绝，只需要在博客仓库下的 `.github/workflows/pages-deploy.yml` 中更改如下即可

```yaml
      - name: Test site
        run: |
          bundle exec htmlproofer _site \
            \-\-disable-external \
            \-\-no-enforce-https \ # 增加这一行
            \-\-ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```

### 修改文章链接格式

默认的文章链接格式为 `xxx.github.io/posts/title`，这也符合常规，但由于我是中文博客，文章标题都是中文，这样在分享链接的时候将会把所有中文字符转义，不仅长而且丑。

打开 `_config.yml`，找到 `defaults` 下属性 `type` 为 `posts` 的 `-scope`，在其 `values` 下有 `permalink` 属性，Chirpy 主题默认为 `/posts/:title/`，我将其修改为 `/posts/:year/:month/:day/:hour/:minute/`，考虑到文章的发表频率，这样的格式是能接受的。如果想改成别的格式可以参考 [Jekyll 的官方文档](https://jekyllrb.com/docs/permalinks/)。

需要注意如果有文章引用了其他文章，需要同步修改链接。

### 修改 Tabs 的名称

Tabs 指博客左边的这一列链接，其中的归档（Archives）页面其实是一条时间轴，因此更改其名称为 `时间线`，参考[这个讨论](https://github.com/cotes2020/jekyll-theme-chirpy/discussions/1264)。

如果博客配置为英文，那么只要更改目录下的 `_tabs/archives.md` 名称为 `timeline.md` 即可，注意其内容不需要更改。如果配置为中文，那么需要同步更改本地化资源文件 `_data/locales/zh-CN.yml`，该文件的获取已在[更改版权信息](#更改版权信息)中说明。将其中 `tabs` 下的 `archives: 归档` 更改为 `timeline: 时间线`。

### 自定义 404 页面

从 [Chirpy 主题的仓库](https://github.com/cotes2020/jekyll-theme-chirpy/blob/master/assets/404.html)中复制 `404.html` 到 `assets/` 下，修改如下：

```html
---
layout: page
title: "404: Page not found"
permalink: /404.html

redirect_from:
- /norobots/
- /assets/
- /posts/
---

{% include lang.html %}

<p class="lead">{{ site.data.locales[lang].not_found.statement }}</p>

<script>
    setTimeout(function () {
        window.location.href = "/";
    }, 3000);
</script>
```

首先增加了 `/norobots/`、`/assets/`、`/posts/` 的链接跳转，以防止直接访问目录页面，跳转需要 `jekyll-redirect-from` 插件支持。首先在根目录下的 `Gemfile` 文件中加上 `gem "jekyll-redirect-from"`，然后在根目录下的命令行中运行 `bundle install` 以安装插件，最后在 `_config.yml` 中添加：

```yaml
plugins:
  - jekyll-redirect-from
```

即可。

然后增加了定时跳转回主页的功能，通过内嵌 js 代码实现。

最后如果要修改 404 页面提示词的话，需要修改 `_data/locales/zh-CN.yml` 中的 `not_found: statement:`。

## 如何撰写文章

Jekyll 是一个主要使用 Markdown 文件生成静态网页的框架，因此博客文章的撰写都是使用 Markdown 完成，[参考教程](https://chirpy.cotes.page/posts/write-a-new-post/#mermaid)。

Markdown 文件存放在 `_posts` 下，标题需要满足 `YYYY-MM-DD-TITLE.EXTENSION` 的格式，拓展名可以是 `.md` 或 `.markdown`。内容上与常规 Markdown 的主要区别就是多了一个 Front Matter 的头部块用于标注元信息，模板如下：

```markdown
---
title: "这是一篇新博客"
description: "介绍了如何撰写文章"
date: 2026-04-01 20:23:00 +0800
categories: [教程]
tags: [Jekyll, Chirpy]
copyright:
image: /assets/img/2026/goto_harmful_image.png
math: false
mermaid: false
pin: false
---
```

- `title`：即文章标题，使用英文双引号包括以防止识别到特殊符号
- `description`：文章描述，为空时需要加一对双引号，否则会导致 Jekyll 自动摘取正文开头作为描述
- `date`：文章的发布日期，精确到分，需要在末尾指定时区
- `categories`：文章的分类，最多只能有2个，**英文逗号分隔**
- `tags`：文章的标签，可以有无限多个，**英文逗号分隔**
- `copyright`：单独指定文章版权信息时使用，默认时则为空，或者删除这个属性
- `image`：文章的封面图片，后面为图片的地址，不需要就留空或者删除这个属性
- `math`：是否启用数学公式
- `mermaid`：是否启用图标功能
- `pin`：是否在主页置顶

需要注意的是 Front Matter 中的 title 具有一级标题的作用，因此原先 Markdown 中的一级标题就可以删去。

文章的资源文件如图片等，单独放在 `assets/` 下，建议以年份分隔文件夹：

```
assets
|- file
|  |- 2021
|  |- 2022
|- img
   |- 2021
   |- 2023
   |- favicons
```

如果你对文章进行修改的话，你会发现博客文章开头会标注发布时间和修改时间，这个是通过什么做到的呢？

首先发布时间是自己指定的，而修改时间是通过插件 `posts-lastmod-hook.rb` 在编译时通过 Git 历史信息检测得知的，如果文件有过修改那么就会显示修改时间。

Jekyll 除了常规的 Markdown 还支持一些拓展，参考[这里](https://chirpy.cotes.page/posts/write-a-new-post/)。
