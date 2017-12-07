---
layout:     post
title:      SSO支持docker运行啦
summary:    新特性docker快速运行
tags:       [Blog]
---

# SSO新特性

> 经过折腾，终于支持docker运行
目前仅仅支持`config-server`、`sso-server`在容器下运行，问题也比较多，镜像庞大，配置不能持久化，其他服务尚未加入等等再以后慢慢解决

docker 快速运行

```cmd
docker run -d --restart=always -p 8443:8443 kawhii/sso 
```