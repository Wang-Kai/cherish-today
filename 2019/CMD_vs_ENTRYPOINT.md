# Dockerfile 中 CMD & ENTRYPOINT 指令


### CMD

> The CMD instruction has three forms:
> 
>- CMD ["executable","param1","param2"] (exec form, this is the preferred form)
>- CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
>- CMD command param1 param2 (shell form)
>
There can only be one CMD instruction in a Dockerfile. If you list more than one CMD then only the last CMD will take effect.

>The main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well.

官方文档解释：

- **在整个 Dockerfile 中 CMD 命令只能出现一次**，若出现多次，则使用最后一个
- CMD 指令的主要目的是对可执行容器提供默认的命令
- 只提供 `参数` 没有 `执行命令` 的 `CMD` 需要 `ENTRYPOINT` 配合使用，其提供的参数将作为 ENTRYPOINT 的默认参数（如果在命令行运行时提供了参数，则 CMD 参数将被忽略）

### ENTRYPOINT 

>ENTRYPOINT has two forms:

>- ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred)
>- ENTRYPOINT command param1 param2 (shell form)
>An ENTRYPOINT allows you to configure a container that will run as an executable.

ENTRYPOINT 使得容器变得可执行

> You can specify a plain string for the ENTRYPOINT and it will execute in /bin/sh -c. This form will use shell processing to substitute shell environment variables, and will ignore any CMD or docker run command line arguments. 

使用 shell 模式的话将忽略 CMD 和 运行容器时的参数。

> You can use the exec form of ENTRYPOINT to set fairly stable default commands and arguments and then use either form of CMD to set additional defaults that are more likely to be changed.

使用 exec 模式的话命令和参数都是稳定不变的，CMD 提供的参数将作为默认参数

### Summary

1. 一个 Dockerfile 文件中至少指明 CMD or ENTRYPOINT 其中之一