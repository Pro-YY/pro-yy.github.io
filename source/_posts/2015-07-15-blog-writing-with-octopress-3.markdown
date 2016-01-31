---
layout: post
title: "Blog Writing with Octopress 3"
date: 2015-07-15 20:38:31 +0800
comments: true
categories:
---

## initialize workspace

```
  git clone git@github.com:Pro-YY/pro-yy.github.io.git -b source
  cd pro-yy.github.io/
  git clone git@github.com:Pro-YY/pro-yy.github.io.git -b master _deploy
  bundle install
```
## write post

```
  rake new_post["Blog Writing with Octopress 3"]
  vim source/_posts/2015-07-15-blog-writing-with-octopress-3.markdown
```

## preview

```
  rake generate
  rake preview
```

preview at http://localhost:4000

## deploy

```
  rake deploy
```

## commit source

```
  git commit -asm 'new post'
  git push
```

## update octopress
```
  git remote add octopress git://github.com/imathis/octopress.git
  git pull octopress master
  bundle install
  rake update_source
  rake update_style
```
