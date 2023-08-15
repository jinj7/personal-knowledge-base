
# 1. strace

trace system calls and signals

```sh
strace -T -tt -p PID -e trace=network -o 1.txt
```

&nbsp;


# 2. sar

Collect, report, or save system activity information

```sh
sar -n DEV 1
```

&nbsp;


# 3. iperf

## 3.1 UDP test

- server

    ```sh
    iperf -u -s -p 1111 -B 192.168.0.42
    ```

- client

    ```sh
    iperf -u -c 192.168.0.25 -p 1111 -b 100M -t 300 -i 2
    ```
  
    -b 100M  #用于udp测试时，设置测试发送的带宽，单位：bit/秒，不设置时默认为：1Mbit/秒

&nbsp;


# 4. ps

- getting startup time of process

    ```sh
    ps -eo pid,lstart,etime,cmd | grep xxx
    ```

- getting all threads of process

    ```sh
    ps -eLo tid,psr,pcpu,args,comm | awk '$4 ~ /xxx/ {print $0}'
    ```

&nbsp;


# 5. apt & dpkg

- download deb package

    ```sh
    apt-get download xxx
    ```

- list the contents of the deb package

    ```sh
    dpkg-deb -c xxx.deb
    ```

- extract files from the deb package

    ```sh
    dpkg-deb -xv xxx.deb ./tmp
    ```

- query the deb package to which the file belongs

    ```sh
    dpkg -S /path/to/file
    ```

- query the detailed information of the deb package

    ```sh
    dpkg -s xxx
    ```

- query all versions of deb packages

    ```sh
    apt-cache policy xxx
    ```

&nbsp;


# 6. split

split a file into pieces

```sh
split -b 10M data.file -d -a 2 split_file
```

recover from splitted files

```sh
cat split_file* > data.file
```

&nbsp;


# 7. SCP through a proxy server

```sh
scp -o "ProxyJump <User>@<Proxy-Server>" <File-Name> <User>@<Destination-Server>:<Destination-Path>

scp -o "ProxyCommand ssh <user>@<Proxy-Server> nc %h %p" <File-Name> <User@<Destination-Server>:<Destination-Path>
```

&nbsp;


# 8. Echo server with ncat 

```sh
ncat -e /bin/cat -k -u -l 20000
```

&nbsp;
