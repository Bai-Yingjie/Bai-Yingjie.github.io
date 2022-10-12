mdBook是rust写的一个工具, 用来把md文档转成html book.  
guide: https://rust-lang.github.io/mdBook

mdBook本身也是个git repo: https://github.com/rust-lang/mdBook

- [再更新2022.10](#再更新202210)
- [更新 2022.08](#更新-202208)
- [安装](#安装)
- [book组织](#book组织)
- [mdbook使用](#mdbook使用)
  - [book.toml](#booktoml)
  - [SUMMARY.md](#summarymd)
  - [build book](#build-book)

# 再更新2022.10
关键参考: https://www.mapull.com/gitbook/comscore/  
这个site就是用gitbook写的, gitbook的配置写的非常到位!
book.json
```json
{
  "title": "GitBook 简明教程",
  "language": "zh-hans",
  "author": "码谱",
  "links": {
    "sidebar": {
      "码谱": "http://www.mapull.com"
    }
  },
  "plugins": [
    "-search",
    "-lunr",
    "-sharing",
    "-livereload",
    "github",
    "donate",
    "chart",
    "todo",
    "graph",
    "puml",
    "katex",
    "code",
    "ace",
    "sitemap-general",
    "mermaid-gb3",
    "include-csv",
    "flexible-alerts",
    "chapter-fold",
    "anchor-navigation-ex",
    "theme-comscore"
  ],
  "pluginsConfig": {
    "anchor-navigation-ex": {
      "showLevel": false,
      "showGoTop": true
    },
    "sitemap-general": {
      "prefix": "https://www.mapull.com/gitbook/comscore/"
    },
    "my-toolbar": {
      "buttons": [
        {
          "label": "下载PDF",
          "icon": "fa fa-file-pdf-o",
          "url": "https://www.mapull.com/gitbook/comscore/book.pdf",
          "position": "left",
          "text": "下载PDF",
          "target": "_blank"
        }
      ]
    },
    "donate": {
      "wechat": "https://www.mapull.com/logo/mapull-qr.png",
      "button": "反馈",
      "wechatText": "微信扫码"
    },
    "versions": {
      "options": [
        {
          "value": "https://www.mapull.com/gitbook/api/",
          "text": "Theme API"
        },
        {
          "value": "https://www.mapull.com/gitbook/comscore/",
          "text": "Theme comscore",
          "selected": true
        }
      ]
    },
    "github": {
      "url": "https://gitee.com/mapull/gitbook-guide"
    },
    "edit-link": {
      "base": "https://gitee.com/mapull/gitbook-guide",
      "label": "Edit This Page"
    }
  },
  "variables": {
    "mapull": "码谱，让编程更容易。",
    "ides": [{"name": "Eclipse"}, {"name": "IntelliJ IDEA"}, {"name": "Visual Studio Code"}]
  },
  "structure": {
    "readme": "home.md"
  }
}
```

# 更新 2022.08
mdbook不支持中文搜索, 故弃用. 使用gitbook代替  
gitbook参考:   
https://github.com/zhangjikai/gitbook-use  
https://github.com/snowdreams1006/snowdreams1006.github.io/blob/master/book.json  
https://snowdreams1006.github.io/

# 安装
可以直接去github页面下载: https://github.com/rust-lang/mdBook/releases  
也可以自己编译, 但需要先安装rust编译器
```
cargo install mdbook
```
cargo命令会自动从[crates.io](https://crates.io/)下载mdbook, 编译, 然后安装到cargo的bin目录(默认是`~/.cargo/bin/`).

crates.io上的版本会比github代码稍微滞后一点, 可以指定用github代码编译:
```
cargo install --git https://github.com/rust-lang/mdBook.git mdbook
```

# book组织
book由chapter组成, 每个chapter是一个独立的page, chapter可以有子chapter.

# mdbook使用
```sh
# 新建一个book
mdbook init my-first-book
cd my-first-book
# 开启一个webserver, 修改的内容可以自动刷新到web page
mdbook serve
```

## book.toml
一个book需要几个特殊文件来定义排版和布局. 根目录下的book.toml就是其中一个:
最常用的, 最简单的:
```toml
[book]
title = "My First Book"
```
mdbook自己的实例:  
https://github.com/rust-lang/mdBook/blob/master/guide/book.toml

## SUMMARY.md
这个文件在src目录下, 定义了chapter结构:
```
# Summary

[Introduction](README.md)

- [My First Chapter](my-first-chapter.md)
- [Nested example](nested/README.md)
    - [Sub-chapter](nested/sub-chapter.md)
```
实例:
https://github.com/rust-lang/mdBook/blob/master/guide/src/SUMMARY.md

## build book
```
mdbook build
```
这个命令会根据md文件在本地book目录下生成html.