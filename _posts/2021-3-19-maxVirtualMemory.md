---
title: "Maximum Virtual Memory per Linux Process"
categories:
  - Uncategorized
---

A small test.

## Environment

* WSL2 Ubuntu 20.04 (*could VM be a problem?*)
* DELL XPS15 with 20GB physical memory and 512GB NVME SSD

## Check Out Upper Limit Settings

Use the Linux *ulimit* command:

```
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 63170
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 63170
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

## The Test Program

A naive test, *never tried* to operate on any of the allocated memory.

```c++
#include <iostream>
using namespace std;
#include <cstdio>

int cnt = 0;

int main() {
    while(1) {
        int* ptr = (int*)malloc(sizeof(int) * (1<<25));
        if(ptr == NULL) {
            break;
        } else {
            cnt++;
        }
        if(cnt % 10 == 0) {
            printf("%d\n", cnt);
        }
    }
    return 0;
}
```

## Result

A total of 15.2TB memory is acquired. Meanwhile, disk IO is 100% saturated, indicating intensive swaps.

```
486910
486920
486930
486940
486950
486960
486970
486980
486990
487000
Killed
jing@DESKTOP-8QT2VLO:~/code$ 
```