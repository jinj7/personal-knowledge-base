
# 1. regular expression

```bash
#!/bin.bash

if [[ $# -lt 2 ]]; then
    echo "Usage: $0 PATTERN STRINGS..."
    exit 1
fi
regex=$1
shift
echo "regex: $regex"
echo

while [[ $1 ]]
do
    if [[ $1 =~ $regex ]]; then
        echo "$1 matches"
        i=1
        n=${#BASH_REMATCH[*]}
        while [[ $i -lt $n ]]
        do
            echo "  capture[$i]: ${BASH_REMATCH[$i]}"
            let i++
        done
    else
        echo "$1 does not match"
    fi
    shift
done
```

| array member | descriptions |
| :---  | :---  |
| BASH_REMATCH[0] | entire match |
| BASH_REMATCH[1] | first sub-pattern |
| BASH_REMATCH[2] | second sub-pattern |

References:
[Bash Regular Expression](https://www.linuxjournal.com/content/bash-regular-expressions)

&nbsp;


# 2. associative array

An array variable is used to store multiple data with index and the value of each array element is accessed by the corresponding index value of that element. The array that can store string value as an index or key is called associative array. An associative array can be declared and used in bash script like other programming languages.

## 2.1 declare and intialize

```bash
declare -A assArray1
assArray1[fruit]=Mango

declare -A assArray2=( [HDD]=Samsung [Monitor]=Dell [Keyboard]=A4Tech )
```

## 2.2 access

```bash
echo ${assArray1[fruit]}

# print all keys
for key in "${!assArray1[@]}"; do echo $key; done
echo "${!assArray1[@]}"

# print all values
for val in "${assArray1[@]}"; do echo $val; done
echo "${assArray1[@]}"

# print all keys and values
for key in "${!assArray1[@]}"; do echo "$key => ${assArray1[$key]}"; done
```

## 2.3 find missing index

```bash
if [ ${assArray2[Monitor]+_} ]; then echo "Found"; else echo "Not found"; fi
```

References:
[Associative array in Bash](https://linuxhint.com/associative_array_bash/)

&nbsp;
