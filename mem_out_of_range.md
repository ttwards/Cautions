## 今天写SBUS驱动内存越界了
### 问题代码
```c
int offset = 0;
        while ((offset = find_begin(p, len)) != -1) {
            p += offset + 25;
            len -= offset + 25;
        }
        if (len > 25) {
            memcpy(rx_date, p, 25); //这实际上是从sbus帧的结尾开始复制，可能发生越界，一般会修改到dma的缓冲区导致fault
            sbus_parseframe();
        }
```
```c
static int find_begin(const uint8_t *data, uint16_t len, uint16_t start) {
    if (len < 25) {
        return -1;
    }
    for (int i = str; i < len; i++) {
        if (data[i] == 0x0F && data[i + 24] == 0x00) {
            return i;
        }
    }
    return -1;
}
```
### 错误信息
```
[00:00:00.205,000] <err> os: ***** USAGE FAULT *****
[00:00:00.205,000] <err> os:   Unaligned memory access
[00:00:00.205,000] <err> os: r0/a1:  0x0000046d  r1/a2:  0x00000000  r2/a3:  0x00000000
[00:00:00.205,000] <err> os: r3/a4:  0x5555556e r12/ip:  0x00000018 r14/lr:  0x08018aff
[00:00:00.205,000] <err> os:  xpsr:  0x2c2584b0
[00:00:00.205,000] <err> os: Faulting instruction address (r15/pc): 0x9612c30e
[00:00:00.205,000] <err> os: >>> ZEPHYR FATAL ERROR 31: Unknown error on CPU 0
[00:00:00.205,000] <err> os: Fault during interrupt handling

[00:00:00.205,000] <err> os: Current thread: 0x200007a0 (idle)
[00:00:00.205,000] <err> os: Halting system
```
或者
```
[00:00:00.056,000] <err> os: ***** BUS FAULT *****
[00:00:00.061,000] <err> os:   Precise data bus error
[00:00:00.067,000] <err> os:   BFAR Address: 0x363920
[00:00:00.073,000] <err> os: r0/a1:  0x20000594  r1/a2:  0x2000405c  r2/a3:  0x00363920
[00:00:00.081,000] <err> os: r3/a4:  0x00363920 r12/ip:  0xaaaaaaaa r14/lr:  0x0800ad4d
[00:00:00.090,000] <err> os:  xpsr:  0x21000000
[00:00:00.095,000] <err> os: Faulting instruction address (r15/pc): 0x0800bdd0
[00:00:00.103,000] <err> os: >>> ZEPHYR FATAL ERROR 25: Unknown error on CPU 0
[00:00:00.111,000] <err> os: Current thread: 0x20000960 (sysworkq)
[00:00:00.118,000] <err> os: Halting system
```
### 查错过程
#### 1. 根据pc寄存器`800bdd0`反编译找到出错位置
```
objdump -S build/zephyr/zephyr.elf > assembly.txt
```
```
; 	return k;
 800bdc0: 697b         	ldr	r3, [r7, #0x14]
 800bdc2: 61fb         	str	r3, [r7, #0x1c]
; 	if (slab->free_list != NULL) {
 800bdc4: 68fb         	ldr	r3, [r7, #0xc]
 800bdc6: 68db         	ldr	r3, [r3, #0xc]
 800bdc8: 2b00         	cmp	r3, #0x0
 800bdca: d010         	beq	0x800bdee <k_mem_slab_alloc+0x6e> @ imm = #0x20
; 		*mem = slab->free_list;
 800bdcc: 68fb         	ldr	r3, [r7, #0xc]
 800bdce: 68da         	ldr	r2, [r3, #0xc]
 800bdd0: 68bb         	ldr	r3, [r7, #0x8]
 800bdd2: 601a         	str	r2, [r3]
; 		slab->free_list = *(char **)(slab->free_list);
 800bdd4: 68fb         	ldr	r3, [r7, #0xc]
 800bdd6: 68db         	ldr	r3, [r3, #0xc]
 800bdd8: 681a         	ldr	r2, [r3]
 800bdda: 68fb         	ldr	r3, [r7, #0xc]
 800bddc: 60da         	str	r2, [r3, #0xc]
; 		slab->info.num_used++;
 800bdde: 68fb         	ldr	r3, [r7, #0xc]
 800bde0: 699b         	ldr	r3, [r3, #0x18]
 800bde2: 1c5a         	adds	r2, r3, #0x1
 800bde4: 68fb         	ldr	r3, [r7, #0xc]
 800bde6: 619a         	str	r2, [r3, #0x18]
; 		result = 0;
```
发现是申请缓存的时候报错，查找附近代码后发现问题
修改：
```c
uint8_t *ptr_offset = p + find_begin(p, len);
if (len > 25) {
        memcpy(data->data, ptr_offset, 25);
}
```
```c
static uint8_t find_begin(const uint8_t *data, uint16_t len) {
    if (len < 25) {
        return -1;
    }
    for (int i = len - 1; i >= 0; i--) {
        if (data[i - 24] == 0x0F && data[i] == 0x00) {
            return i - 24;
        }
    }
    return -1;
}
```
