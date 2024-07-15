### 创建镜像命令
```
docker build -t diy:1 .
```

### 部分源码修改自
https://github.com/Axizdkr/tengine/tree/master

tengine + acme.sh 二合一 镜像 tengine-acme.sh
用 supervisor 管理 nginx 和 cron

> 有缺点：违背了docker容器只运行一个程序的初心，再一个docker容器中运行多个程序，会让容器反应变慢，就拿docker restart tacme 重启容器来说，重启耗时很长。

### 不推荐的做法， tengine + acme.sh docker 容器
```sh
docker run -itd \
-p 80:80 \
-p 443:443 \
-v /root/out:/acme.sh \
-v /root/ssl:/ssl \
--name tacme \
tengine-acme.sh
```

### 推荐的做法，nginx与acme.sh各自分开，单独以 docker 守护进程运行 acme.sh 程序
```sh
docker run --rm  -itd \
  -v "$(pwd)/out":/acme.sh \
  -v "$(pwd)/ssl":/ssl \
  --net=host \
  --name=tacme \
  neilpang/acme.sh daemon
```

### 测试 acme.sh 是否正常运行
```sh
docker exec tacme acme.sh -v
#https://github.com/acmesh-official/acme.sh
#v3.0.8
```

### 绑定一个邮箱，不然会提示无法生成证书
#acme.sh 默认是使用 zerossl 颁布证书，需要绑定邮箱。

#如果切换成使用 letsencrypt 颁布证书就不需要绑定邮箱。

#绑定邮箱后，申请的证书记录可以在web页面 https://app.zerossl.com/certificates/issued 查看
```sh
docker exec tacme acme.sh --register-account -m abc@qq.com
```

### 申请ssl证书
#因为80端口被nginx占用，所以 `--standalone` 模式下要指定另外的端口 `--httpport 81`

#这里申请的是 一级域名 abc.com 和 二级域名 www.abc.com 的多域名证书。

#如果你在nginx中配置了 https://abc.com 跳转到 https://www.abc.com 则 二级域名 www.abc.com 不能缺少。
```sh
docker exec tacme acme.sh --issue \
-d abc.com \
-d www.abc.com \
--standalone \
--httpport 81
```

### 指定容器web页路径
#如果出现超时`Timeout`或者`404`找不到https://www.abc.im/.well-known/acme-challenge/xxxxxx 文件

#这种情况一般出现在，你的nginx环境已经搭建好,nginx配置文件也已经配置好。如果只是acme.sh单容器，使用`--standalone` 模式就可以成功

#需要你指定 `-w /etc/nginx/html/abc.im` 容器web路径
```sh
docker exec tacme acme.sh --issue \
  -d abc.im \
  -d www.abc.im \
  -w /etc/nginx/html/abc.im \
  --keylength ec-256
```

### 复制证书到指定目录(此目录必须已经存在)
```sh
docker exec tacme acme.sh --install-cert \
-d abc.com \
--key-file       /ssl/abc.im/key.pem \
--fullchain-file /ssl/abc.im/fullchain.pem \
--cert-file      /ssl/abc.im/cert.pem \
--ca-file        /ssl/abc.im/ca.pem
```

### 切换默认证书申请服务器 zerossl 或 letsencrypt
```sh
docker exec tacme acme.sh --set-default-ca --server letsencrypt
docker exec tacme acme.sh --set-default-ca --server zerossl
```

### 查看容器中定时任务
```sh
crontab -l
```
> 4 2 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" --config-home "/acme.sh" > /proc/1/fd/1 2>/proc/1/fd/2
