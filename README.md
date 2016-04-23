# andyos notes

* [0. 工具](#id0)

* [1. Bochs引导软盘试验](#id1)

<a id="id0"></a>
## 0. 工具 

* [Bochs x86 PC emulator](https://sourceforge.net/projects/bochs/files/bochs/)
	下载最新版本，编译（开启debugger和disasm），安装。
```bash
tar -vxzf bochs-2.6.8.tar.gz
cd bochs-2.6.8
./configure --enable-debugger --enable-disasm
make
sudo make install
```

* nasm 汇编器，这里`build-essential`包括GCC和GNU-Make。
```bash
sudo apt-get install build-essential nasm
```

<a id="id1"></a>
## 1. Bochs引导软盘试验

从结果上看，执行命令行
```bash
bochs
```
即运行 PC 模拟器， 并能成功引导名为 a.img 的软盘映像文件。
引导程序什么也不做，只是死循环。

bochs 之所以能够引导 a.img ，是因为 bochsrc 文件。
bochsrc文件如下：
```nasm
# how much memory the emulated machine will have
megs: 32

# filename of ROM images
romimage: file=/usr/local/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/share/vgabios/vgabios.bin

# what disk images will be used
floppya: 1_44=a.img, status=inserted

# choose the boot disk
boot: floppy
```
这里 bochsrc 文件设置了 PC 模拟器的内存大小，ROM 的 image 文件，外存的软盘 image 文件，以及启动盘。
`boot: floppy` 设置了 PC 模拟器从软盘启动。
`floppya: 1_44=a.img, status=inserted` 设置了 PC 模拟器的连接软盘 A，
且软盘 A 的映像文件为当前目录下的 a.img。

过程如下：PC 模拟器的 BIOS 程序根据配置从软盘启动系统，它搜索全部已插入的软盘且发现软盘 A，
确认软盘 A 的第 0 号扇区的最后两个字节为 `0xAA55`，以此认为软盘 A 是系统引导盘。
BIOS 程序加载第 0 号扇区到内存中（起始地址为`0x7c00`），并将控制权交给这段程序。
至此，BIOS 程序完成使命，引导扇区程序开始其使命。

上述过程的关键字：第 0 号扇区，`0xAA55`，`0x7c00`。


如果通过软盘引导系统，BIOS 将加载以 `0xAA55` 结尾的0号扇区到地址 `0x7c00` 开始的内存中，
并将控制权交给该扇区中的指令。

boot.asm 文件仅包含以下一个语句，表示一旦进入即死循环。
```nasm
	jmp	$
```

为了顺利完成引导试验，还需要创建两个文件：bochsrc文件和虚拟软盘映像文件。
虚拟软盘映像文件，取名为a.img，可以通过简单的 c 程序创建。如下：
```c
/*
 * aimg.c
 * 创建 a.img 软盘虚拟映像文件，包含Boot Sector，大小为1.44MB
 */
#include <stdio.h>

int main()
{
	int i;
	int len;
	FILE* fp;
	char buffer[512];
	char bufferz[512];
	for (i=0; i<512; i++) {
		bufferz[i] = 0;
	}
	
	/* 以二进制只读模式打开 "boot.bin" 文件 */
	fp = fopen("boot.bin", "rb");
	if (!fp) {
		printf("fail to open \"boot.bin\"!\n");
		return 0;
	}

	/* 读取文件大小，确保小于510字节 */
	fseek(fp, 0, SEEK_END);
	len = ftell(fp);
	fseek(fp, 0, SEEK_SET);
	if (len > 510) {
		printf("\"boot.bin\" is too large!\n");
		fclose(fp);
		return 0;
	}
	
	/* 读取 "boot.bin" 文件，关闭文件 */
	fread(&buffer, len, 1, fp);
	fclose(fp);

	/* 以二进制写模式创建 "a.img" 文件 */
	fp = fopen("a.img", "wb");	
	if (!fp) {
		printf("fail to create \"a.img\"!\n");
		return 0;
	}

	/* 写 "a.img" 文件，以0xAA55结尾，填充512字节 */
	fwrite(&buffer, len, 1, fp);
	fwrite(&bufferz, 510-len, 1, fp);
	buffer[0] = 0x55;
	buffer[1] = 0xAA;
	fwrite(&buffer, 2, 1, fp);
	fclose(fp);

	return 0;
}
```

makefile 文件如下：
```makefile
all:
	nasm -o boot.bin boot.asm
	gcc -o aimg aimg.c
	./aimg

.PHONY: clean
clean:
	rm -f aimg boot.bin a.img

```

执行如下命令，即可看到Bochs 顺利引导虚拟软盘映像文件：
```bash
make
bochs
```

## 2. 完成引导扇区

```
1. boot.asm   -> boot.bin :
	nasm -o boot.bin boot.asm

2. kernel.asm -> kernel.bin :
	nasm -f elf -o kernel.o kernel.asm
	ld -m elf_i386 -s -Ttext 0x30400 -o kernel.bin kernel.o

3. aimg.c     -> aimg :
	gcc -o aimg aimg.c

4.            -> a.img :
	./aimg
```

BIOS 加载 0 扇区的 512 字节到内存 `0x7c00` 处，并从这里开始执行启动程序。
体现在代码中：
```nasm
org	0x7c00
```

**Boot Sector 的任务有两个：加载 kernel.bin 和进入保护模式。**

加载 kernel.bin，即将 kernel.bin 从外存（软盘）读取写入内存的过程。
这个功能可以通过 BIOS 的 13 号终端实现：

| 寄存器 | 含义              |
| :----: | :---------------- |
| ah     | 2                 |
| al     | 读取扇区数量      |
| es:bx  | 写入内存位置      |
| ch     | 磁道号，从 0 开始 |
| cl     | 扇区号，从 1 开始 |
| dh     | 磁头号            |
| dl     | 0                 |

这个过程涉及 kernel.bin 在 a.img 中的位置。
依照简单原则，设计如下：

| 扇区号 | 含义           |
| :----: | :---           |
| 0      | Boot Sector    |
| 1      | 首字节数值为 N |
| 2..N+1 | kernel.bin     |
| N+2... | ...            |

加载 kernel.bin 的代码：

```nasm
	mov	ah, 2		;2
	mov	al, 1		;count
	mov	ax, BaseK	
	mov	es, ax
	mov	bx, OffsetK	;BaseK:OffsetK -> 起始内存地址
	mov	ch, 0		;track no
	mov	cl, 2		;sector no
	mov	dh, 0		;head no
	mov	dl, 0		;0
	int	13h

	mov	al, byte [es:bx]
	mov	ah, 2
	mov	cl, 3
	int	13h
```
调用两次 13 号中断完成 kernel.bin 的加载。


kernel.bin 是 elf 格式的二进制指令，包含elf header，program header table, section table 和 section header table。
这里我们只关注elf header 和 program header table 两个结构。

elf header：

| 偏移 | 长度 | 含义      |
| :--: | :--: | :------   |
| 0    | 16   | identity  |
| 16   | 2    | type      |
| 18   | 2    | machine   |
| 20   | 4    | version   |
| 24   | 4    | **entry** |
| 28   | 4    | **phoff** |
| 32   | 4    | shoff     |
| 36   | 4    | flags     |
| 40   | 2    | ehsize    |
| 42   | 2    | phentsize |
| 44   | 2    | **phnum** |
| 46   | 2    | shentsize |
| 48   | 2    | shnum     |
| 50   | 2    | shstrndx  |

其中 entry 是入口地址，各个 .o 模块在链接过程中可以通过 -Ttext 开关指定。
phoff 是 program header table 在文件中的偏移，一般为 0x34，即elf header的长度。
phnum 是 program header 的数量。
根据这三个参数，遍历全部的program header。

program header：

| 偏移 | 长度 | 含义       |
| :--: | :--: | :--------- |
| 0    | 4    | type       |
| 4    | 4    | **offset** |
| 8    | 4    | **vaddr**  |
| 12   | 4    | paddr      |
| 16   | 4    | **filesz** |
| 20   | 4    | memsz      |
| 24   | 4    | flags      |
| 28   | 4    | align      |

其中 offset 是section 在文件中的偏移，vaddr 是section在内存中的位置，filesz 是section的大小。
根据这三个参数，将各个section放置到合适的内存位置。

初始化内核的代码如下：

```nasm
	mov	ax, BaseK	;make sure ds -> BaseK
	mov	ds, ax
	mov	ax, BaseEntry	;es -> BaseEntry
	mov	es, ax

	mov	dx, word [ds:2Ch]	;number of program headers
	mov	bx, [ds:1Ch]		;offset of program header table
loop:
	mov	cx, word [ds:(bx+10h)]	;size = size32 & 0xFFFF, if size32 > 64K then errors occur!
	mov	di, word [ds:(bx+8h)]	;dst = dst32 & 0xFFFF
	mov	si, [ds:(bx+4h)]	;src = src32 & 0xFFFF

	cmp	cx, 0
	jz	.nop
.loop:
	lodsb
	stosb
	loop	.loop
.nop:
	add	bx, 20h			;to next program header
	dec	dx
	cmp	dx, 0
	jnz	loop
```

进入保护模式：
```nasm
	mov	ax, cs
	mov	ds, ax
	lgdt	[ds:GDTPtr]
	cli
	in	al, 92h
	or	al, 10b
	out	92h, al
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax
```

其中 `GDTPtr` 指向的Global Descriptor Table 如下：
```nasm 
LABEL_GDT:		
Descriptor	0, 0, 0
Descriptor	0, 0FFFFFh, DA_CR + DA_32 + DA_LIMIT_4K
Descriptor	0, 0FFFFFh, DA_DRW + DA_32 + DA_LIMIT_4K
Descriptor	0B8000h, 0FFFFh, DA_DRW + DA_DPL3
GDTPtr			dw	$ - LABEL_GDT - 1
			dd	LABEL_GDT
```
