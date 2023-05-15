# 1. 设置源码路径

1. 列示任意函数

   ```sh
   (gdb) l func
   ```

2. 查看$cdir和$cwnd

   ```sh
   (gdb) info source
   ...
   (gdb) pwd
   ```

3. 替换路径

   ```sh
   (gdb) set substitute-path <from_path> <to_path>
   ```

&nbsp;


# 2. Examining memory

```sh
(gdb) x /nfu addr
```

- n, the repeat count

- f, the display format
  x

  d

  u

  o

  t

  a

  c

  f

  s
  i: machine instructions

- u, the unit size
  b: bytes
  h: two bytes
  w: four bytes (default)
  g: eight bytes

References: https://sourceware.org/gdb/onlinedocs/gdb/Memory.html
