---
layout:     post
title:      "配置zsh终端打开VsCode"
date:       2020-04-06
author:     "Allen"
header-img: "img/code-bg.jpg"
tags:
    - Mac
---

我们在使用zsh终端的时候，如何通过命令快速启动VsCode呢？过程很简单，我们一起看看吧。

首先在iterm2中打开zsh配置文件

`open .zshrc`

然后在弹出的文档末尾处添加如下代码

```
# open vs code

function code {
    if [[ $# = 0 ]]
    then
        open -a "Visual Studio Code"
    else
        local argPath="$1"
        [[ $1 = /* ]] && argPath="$1" || argPath="$PWD/${1#./}"
        open -a "Visual Studio Code" "$argPath"
    fi
}
```

最后执行

`source .zshrc`

使操作立即生效，我们就可以愉快的在命令行执行Vs的打开操作啦。

附终端操作命令

`code`  在终端直接打开VsCode
`code .` 在命令行当前所在目录打开
`code file` 特定打开某某文件，如code 1.txt，即用VsCode打开当前目录下的1.txt文件