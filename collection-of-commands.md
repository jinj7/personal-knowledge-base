
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