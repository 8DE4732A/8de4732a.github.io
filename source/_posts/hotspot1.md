---
title: 编译及调试
date: 2022-06-25 21:00:24
tags:
---
源码: [https://github.com/openjdk/jdk/tree/jdk-11+28](https://github.com/openjdk/jdk/tree/jdk-11+28)

如何编译可以参考源码 doc/building.html

```bash
bash ./configure \
--with-boot-jdk=/home/outman/jdk-10.0.2/ \
--with-target-bits=64 \
--with-debug-level=slowdebug \
--with-native-debug-symbols=internal \
--disable-warnings-as-errors
make
```

### vscode调试

在调试页面新增gdb配置

```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java",
            "args": ["-cp",".","Main"],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

args是要调用的class

![debug](assets/debug.png)

可以看到入口是main.c