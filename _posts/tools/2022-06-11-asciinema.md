---
layout: post
title: asciinema使用笔记
subtitle: asciinema, 一款好用的全平台兼容的Terminal录屏工具使用笔记
cover-img: ["https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122039785.svg"]
thumbnail-img: ["https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122039785.svg"]
tags: [mac, tools, asciinema, svg]
---

# 安装软件
- [asciinema](https://github.com/asciinema/asciinema)
- [svg-term-cli](https://github.com/marionebl/svg-term-cli)

```shell
brew install nodejs
npm install -g svg-term-cli
```


# 生成录屏文件.cast

```shell
asciinema rec a.cast
```

`Ctrl + D 结束录屏`

# 本地播放录屏文件

```shell
asciinema play a.cast
```

# .cast文件转成.svg文件
需要先安装 [svg-term-cli](https://github.com/marionebl/svg-term-cli)

```shell
cat a.cast | svg-term --out a.svg --term iterm2 --profile mymatrix
```

# 配置文件

[$HOME/.config/asciinema/config](https://asciinema.org/docs/config)

