---
layout: code-server
title:  code-server编译安装
date:   2021-4-21 18:01:30
category: "vscode"
---

# code-server编译安装

vscode、code-server

# code-server编译安装

代码下载：
>>git clone https://github.com/cdr/code-server.git

将code-server放在/root/目录下  
```
cd code-server
yarn
yarn watch
```

# FAQ
## yarn 过程中卡到 shellcheck，如下图所示

![image](/images/posts/code-server/1.png)
- 解决办法：  

编辑如下路径文件：
>>vim /usr/local/share/.cache/yarn/v4/npm-shellcheck-1.0.0-263479d92c3708d63d98883f896481461cf17cd0/node_modules/shellcheck/install.sh   

注释掉node download.js | tar "$(tar_options)", 添加两行如下：

![image](/images/posts/code-server/2.png)



./install.sh: line 8: jq: command not found

You are missing a package from your system, to install it just run: sudo apt update && sudo apt install -y jq
