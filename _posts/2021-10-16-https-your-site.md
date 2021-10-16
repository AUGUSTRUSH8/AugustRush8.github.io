---
layout: post
title: 'Halo博客Https一下'
tags: [code]
---

### Halo博客Https一下
今天给 halo 博客配置 https，按照 halo 官方文档使用 certbot 来配置 SSL 证书

```shell
1.安装 certbot 以及 certbot nginx 插件sudo yum install certbot python2-certbot-nginx -y
2.执行配置，中途会询问你的邮箱，如实填写即可sudo certbot --nginx
3.自动续约sudo certbot renew --dry-run
```

