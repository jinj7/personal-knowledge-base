# 1. 本机同步

```sh
rsync -a source destination
```

执行上面的命令后，源目录source被完整地复制到了目标目录destination下面，即形成了destination/source的目录结构。

如果只想同步源目录source里面的内容到目标目录destination，则需要在源目录后面加上斜杠。

```sh
rsync -a source/ destination
```

上面命令执行后，source目录里面的内容，就都被复制到了destination目录里面，并不会在destination下面创建一个source子目录。

有时希望排除某些文件或目录，这是可以用--exclude参数指定排除模式。

```sh
rsync -av --exclude='*.txt' source/ destination
rsync -av --exclude '*.txt' source/ destination
```

--include参数用来指定必须同步的文件模式，往往与--exclude结合使用。

```sh
rsync -av --include='*.txt' --exclude='*' source/ destination
```

&nbsp;


# 2. 远程同步

## 2.1 ssh协议

```sh
rsync -av source username@remote_host:destination
rsync -av username@remote_host:destination source
```

## 2.2 rsync协议

除了使用 SSH，如果另一台服务器安装并运行了rsync守护程序，则也可以用rsync://协议（默认端口873）进行传输。具体写法是服务器与目标目录之间使用双冒号分隔::。

```sh
rsync -av source remote_host::module/destination
```

注意，上面地址中的module并不是实际路径名，而是rsync守护程序指定的一个资源名，由管理员分配。

