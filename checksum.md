
# 1. Principles

[Internet checksum](https://en.wikipedia.org/wiki/Internet_checksum)

&nbsp;


# 2. Fast Update

| Symbols | Meanings |
| :---  | :---  |
| HC | old checksum in header |
| HC' | new checksum in header |
| m | old value of a 16-bit field |
| m' | new value of a 16-bit field |

HC' = ~( ~HC + ~m + m' )

[RFC1624: Computation of the Internet Checksum via Incremental Update](https://datatracker.ietf.org/doc/html/rfc1624)
