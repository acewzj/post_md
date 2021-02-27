---
title: Config_Git
date: 2020-04-06 08:09:03
tags:
- git
- svn
---



这篇文章主要记述了Git、SVN和以及加密传输的一些知识点。

<!--more-->



## GitHub

1.先生成秘钥

ssh-keygen -t rsa -C "[xxxxx@xxxxx.com](mailto:xxxxx@xxxxx.com)"  

\# Generating public/private rsa key pair...

\# 三次回车即可生成 ssh key



1. $ git clone "SSH地址"
2. \#git pull origin master
3. \#git add .
4. \#git commit -m 'init'
5. \#git push origin master
6. git checkout -b ‘name’

## Git配置多个SSH-Key

[SSH Key](https://gitee.com/help/labels/19)

### 背景

当有多个git账号时，比如：

a. 一个gitee，用于公司内部的工作开发；
b. 一个github，用于自己进行一些开发活动；

### 解决方法

1. 生成一个公司用的SSH-Key

```
$ ssh-keygen -t rsa -C 'xxxxx@company.com' -f ~/.ssh/gitee_id_rsa
```

1. 生成一个github用的SSH-Key

```
$ ssh-keygen -t rsa -C 'xxxxx@qq.com' -f ~/.ssh/github_id_rsa
```

1. 在 ~/.ssh 目录下新建一个config文件，添加如下内容（其中Host和HostName填写git服务器的域名，IdentityFile指定私钥的路径）

```
# gitee
Host gitee.com
HostName gitee.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/gitee_id_rsa
# github
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/github_id_rsa
```

4.用ssh命令分别测试

```
$ ssh -T git@gitee.com
$ ssh -T git@github.com
```

## VSCODE 配置SSH

```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa-remote-ssh

ssh-copy-id -i ~/.ssh/id_rsa-remote-ssh acewzj@47.97.25.88
```

https://code.visualstudio.com/docs/editor/variables-reference

------

假设当前workspace的路径为:"C:\Users\admin\Desktop\test"，workspace文件夹下的结构如下（+表示下一层）：

C:\Users\admin\Desktop\test

+.vscode

 ++tasks.json

 ++launch.json

+main.cpp


${workspaceFolder} :表示当前workspace文件夹路径，也即C:\Users\admin\Desktop\test

${workspaceRootFolderName}:表示workspace的文件夹名，也即test

${file}:文件自身的绝对路径，也即C:\Users\admin\Desktop\test\.vscode\launch.json

${relativeFile}:文件在workspace中的路径，也即.vscode\launch.json

${fileBasenameNoExtension}:当前文件的文件名，不带后缀，也即launch

${fileBasename}:当前文件的文件名，launch.json

${fileDirname}:文件所在的文件夹路径，也即C:\Users\admin\Desktop\test\.vscode

${fileExtname}:当前文件的后缀，也即.json

${lineNumber}:当前文件光标所在的行号

${env:PATH}:系统中的环境变量

```
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) 启动",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "g++ build active file",
            "postDebugTask": "delete output file",
            "miDebuggerPath": "/usr/bin/gdb"
        }
    ]
}
```

```
{
    // 有关 tasks.json 格式的文档，请参见
        // https://go.microsoft.com/fwlink/?LinkId=733558
        "version": "2.0.0",
        "tasks": [
            {
                "type": "shell",
                "label": "gcc build active file",
                "command": "/usr/bin/gcc",
                "args": [
                    "-g",
                    "${file}",
                    "-o",
                    "${fileDirname}/${fileBasenameNoExtension}"
                ],
                "options": {
                    "cwd": "/usr/bin"
                },
                "problemMatcher": [
                    "$gcc"
                ],
                "group": {
                    "kind": "build",
                    "isDefault": true
                }
            },
            {   //删除二进制文件
                "type": "shell",
                "label": "delete output file",
                "command": "rm",
                "args": [
                    "${fileDirname}/${fileBasenameNoExtension}"
                ],
                "presentation": {
                    "reveal": "silent", //删除过程不切换终端（专注程序输出）
                }
            }
        ]
    }
```

