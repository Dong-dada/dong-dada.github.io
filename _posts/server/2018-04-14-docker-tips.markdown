---
layout: post
title:  "Docker 小知识"
date:   2018-04-14 18:27:30 +0800
categories: server
---

* TOC
{:toc}


## 在容器中运行 shell

容器启动后以后台运行，有时候需要到容器里看看情况，可以通过 `docker exec` 在容器里打开 shell:

```
docker container exec -t -i mynginx /bin/bash
```

命令执行完之后就会进入到容器的 shell 里, 执行 `exit` 命令可以退出 shell.