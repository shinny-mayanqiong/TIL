# pip-安装到哪里了

## 查看一个 python 安装到哪里了

```bash
$ python3 -m pip show tqsdk
```

## 查看路径
    
```bash
$ python3 -m site 

USER_BASE: '/home/user/.local' (exists)
USER_SITE: '/home/user/.local/lib/python3.9/site-packages' (exists)
ENABLE_USER_SITE: True


$ python3 -m site --user-site

用户的 site-packages 目录位于:

/home/user/.local/lib/python3.9/site-packages
```

## 使用 sudo 安装有什么不同

```bash
$ sudo python3 -m pip show tqsdk

使用 sudo 安装的包位于全局的 site-packages 目录下:

/usr/lib/python3/dist-packages
```

dist-packages 是 Debian 特别约定。

