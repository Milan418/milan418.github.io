---
title: Hexo写文章
date: 2018-03-30 22:45:17
tags: 其它
categories: 其它
description: Hexo写文章时遇到的一些小纠结&解决办法
keywords: Hexo
---

之所以不用博客园之类的写博客，很大部分原因是因为 代码粘贴出来有时很难看，轻度强迫症者有点受不了，所以一直没有坚持下来。
Hexo搭建博客后，采用Markdown语法，我用vs code这个代码编辑器，添加Markdown Preview插件后，整个流程瞬间666了，嘻嘻。

以下记录一些过程中遇到的小纠结点&解决方案，不时更新。

### 如何插入图片

1._config.yml中 post_asset_folder:true
2.安装上传本地图片的插件

```shell
npm install hexo-asset-image --save
```
3.在/source/_posts文件中新建一个与文章xxx.md同名的文件夹xxx
4.markdown格式插入本地图片

```markdown
![img的替代文字](xxx/图片名.jpg)
```
### 引用自己写的文章

``` markdown
 {% post_link 文章文件名（不要后缀） 文章标题（可选） %}
```