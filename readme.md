# 内网搭建 apt 缓存镜像服务器

当你需要频繁地使用 apt 安装软件时，无论你的网多好，都很“慢”。

在内网搭建一个缓存服务器，就非常非常快了。

毕竟，外网的网速是以 MB 计算，而内网的网速是以 GB 计算。



总体思路：用 apt-cacher-ng 搭建 apt 缓存服务器；通过修改客户端的 sources.list 使用缓存服务器

操作系统版本：debian 12.7



> NB
>
> apt-cacher-ng 只缓存请求的文件，你没请求的，它不会缓存。
>
> 如果你想要搭建一个完整的 apt 镜像站。
>
> 你可以使用 apt-mirror
>
> https://github.com/apt-mirror/apt-mirror

# 缓存服务器搭建

## 安装 apt-cacher-ng

```
sudo apt-get update && sudo apt-get install apt-cacher-ng -y
```

## 修改配置文件

我使用的配置文件，放在了这个仓库里。

你可以在此基础上修改配置，或者不改直接用。

```
# 先备份原始配置文件
sudo mv /etc/apt-cacher-ng/acng.conf ~/acng.conf.bk
# 下载配置文件
git clone --depth=1 https://github.com/nealhallyoung/iacs.git && cd iacs
# 移动配置文件
# 你也可以做一些修改，我会给出一些关键点
# mv acng.conf /etc/apt-cacher-ng/acng.conf
```

### 关键点

```
# 缓存文件夹地址，你想把缓存文件存储到什么地方
CacheDir: /var/cache/apt-cacher-ng

# 服务端口
Port:3142

# 设置缓存文件过期时间，单位 天。默认为 4 天
# 我设置成了 30 天过期，你要根据自己的安全需求来改
ExThreshold: 30

# The number can have a suffix (k,K,m,M for Kb,KiB,Mb,MiB)
# 缓存多少文件时，会清理过期文件
# 我设置成 10GiB 
ExStartTradeOff: 10240M
```

### 更高级的用法

apt-cacher-ng 有很多其他功能

文档

https://www.unix-ag.uni-kl.de/~bloch/acng/

你可以去阅读相关内容

## 启动服务器

```
sudo systemctl enable apt-cacher-ng
sudo systemctl restart apt-cacher-ng
# 查看状态
systemctl status apt-cacher-ng
# 查看日志
journalctl -u apt-cacher-ng
```

打开浏览器访问 http://ip:port

比如我的是 http://192.168.40.133:3142

你会看到管理界面

# 客户端使用

## 修改 sources.list

> NB
>
> 2024 年应该用的都是 https 的镜像站点吧！！！
>
> 如果，你用的是 http 站点，你可以参考官方文档做一点点修改就可以用。

在这里是关于 https 镜像站的使用。

官方文档 https://www.unix-ag.uni-kl.de/~bloch/acng/html/howtos.html#ssluse

假设

1。你使用阿里云镜像 `https://mirrors.aliyun.com/debian/ `

2。搭建的 apt-cacher-ng 缓存服务器地址 `http://192.168.40:3142` 

你需要把 `https://` 改成`http://192.168.40:3142/HTTPS///`

原本是这样

```
# debian 12
deb https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
# deb-src https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian-security/ bookworm-security main
# deb-src https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
# deb-src https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
# deb-src https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
```

改成这样

```
# debian 12
deb deb http://192.168.40.133:3142/HTTPS///mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
# deb-src https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib

deb deb http://192.168.40.133:3142/HTTPS///mirrors.aliyun.com/debian-security/ bookworm-security main
# deb-src https://mirrors.aliyun.com/debian-security/ bookworm-security main

deb deb http://192.168.40.133:3142/HTTPS///mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
# deb-src https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib

deb deb http://192.168.40.133:3142/HTTPS///mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
# deb-src https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
```

> NB
>
> 如果你除了 /etc/apt/sources.list 还有其他的 apt 源文件，你也可以相应地做修改

## 清理 apt 缓存

```
sudo rm -rf /var/cache/apt/*  && sudo rm -rf /var/lib/apt/lists/*
sudo apt-get clean
```

## 测试

准备两台机器

在 A 机器上先安装

```
# 这三个都是比较大的软件
sudo apt install build-essential
sudo apt install libreoffice
sudo apt install wine
```

在 B 上再次安装

如果没问题，下载速度，应该非常快。
