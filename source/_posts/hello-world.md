---
title: Hello World
---

第一篇博客 记录 Hexo + NexT搭建个人博客的过程

1. 安装[Hexo](https://hexo.io/)并初始化项目
``` bash
$ npm install -g hexo-cli
$ hexo init <project-name> # 这里我使用blog
```

2. 安装[NexT](http://theme-next.iissnan.com)主题
``` bash
$ cd <project-name> # cd blog
$ git clone https://github.com/hexo-theme/hexo-theme-next themes/next
```
修改顶层 _config.yml:
theme: landscape  
=> theme: next

3. 启动博客 如果一切顺利可以看到 这篇博客下部的内容: Welcome to [Hexo](https://hexo.io/)!
``` bash
$ hexo s
```

4. 发布博客到github page
修改顶层 _config.yml deploy部分
安装 hexo-deployer-git
```bash
$ npm install hexo-deployer-git --save
```
```
deploy:
  type: git
  repo: https://github.com/jimuyang/jimuyang.github.io
  # https://github.com/<username>/<project>
  # branch: gh-pages
```
deploy到github page
```bash
$ hexo clean && hexo deploy
```

Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)
