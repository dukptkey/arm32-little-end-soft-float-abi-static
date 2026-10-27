# arm32-little-end-soft-float-abi-static

This repo is a initial collection of pre-compiled tools for armv7l (ARm 32 bits), litte-endian, soft-float code generated, statically compiled. 

## kread
`kread` is a simple tool to facilitate reading memory from a given address, useful to play with MMIO addresses. Its code is below:

```c
//read kernel memory via /var/kmem                                          
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
int main(int argc, char **argv) {
    unsigned long addr = strtoul(argv[1], 0, 0);
    int n = argc > 2 ? atoi(argv[2]) : 4;
    char *dev = argc > 3 ? argv[3] : "/var/mem";
    int fd = open(dev, O_RDONLY | O_SYNC);
    if (fd < 0) { perror("open"); return 1; }
    unsigned long page = addr & ~0xFFF;
    unsigned long off = addr & 0xFFF;
    void *map = mmap(0, off + n, PROT_READ, MAP_SHARED, fd, page);
    if (map == MAP_FAILED) { perror("mmap"); return 1; }
    unsigned char *p = (unsigned char*)map + off;
    int i; for (i = 0; i < n; i++) {
        printf("%02x ", p[i]);
        if ((i+1) % 16 == 0) printf("\n");
    }
    if (n % 16) printf("\n");
    munmap(map, off + n);
    return 0;
}
```

## frida-server

`frida-server-17.19.0-arm32-static` is Frida 17.19.0's server, built for ARMv7-A, little-endian, soft-float EABI (`arm-linux-gnueabi`), statically linked and stripped. It uses the QuickJS runtime (V8 is excluded). It was tested with `qemu-arm-static` (`--version` prints `17.19.0`)
