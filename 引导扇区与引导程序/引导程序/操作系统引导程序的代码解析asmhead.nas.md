以下是 `asmhead.nas` 的代码功能梳理，它负责从 **实模式** 切换到 **保护模式**，并加载操作系统的核心模块（`bootpack`）。这段代码通常由引导扇区（如 `ipl10.nas`）加载到内存后执行。

---

### **1. 核心功能概述**
1. **初始化显示模式**  
   - 尝试设置高分辨率（VBE模式），失败则回退到VGA模式。
2. **硬件初始化**  
   - 保存键盘LED状态，禁用中断，启用A20地址线。
3. **切换到保护模式**  
   - 设置全局描述符表（GDT），修改CR0寄存器，进入32位保护模式。
4. **加载内核到内存**  
   - 从磁盘或内存中复制内核代码到指定地址（`BOTPAK`）。
5. **跳转执行内核**  
   - 设置栈指针，跳转到内核入口点。

---

### **2. 代码模块详解**

#### **(1) 显示模式初始化**
```nasm
; 尝试设置VBE显示模式（1024x768x8bit）
MOV AX, 0x9000
MOV ES, AX
MOV DI, 0
MOV AX, 0x4f00      ; VBE功能号：获取控制器信息
INT 0x10
CMP AX, 0x004f      ; 检查是否支持VBE
JNE scrn320         ; 不支持则跳转到默认模式

; 设置VBE模式0x105（1024x768x8bit）
MOV CX, VBEMODE     ; CX=0x105
MOV AX, 0x4f01      ; VBE功能号：获取模式信息
INT 0x10
CMP AX, 0x004f
JNE scrn320

; 验证模式参数（颜色深度、内存布局）
CMP BYTE [ES:DI+0x19], 8  ; 颜色深度=8bit
JNE scrn320
CMP BYTE [ES:DI+0x1b], 4  ; 内存模式=4（直接颜色）
JNE scrn320

; 应用VBE模式
MOV BX, VBEMODE+0x4000    ; 设置模式并保持显示内容
MOV AX, 0x4f02            ; VBE功能号：设置视频模式
INT 0x10
JMP keystatus

; 回退到VGA模式320x200x8bit
scrn320:
MOV AL, 0x13         ; VGA模式号0x13
MOV AH, 0x00
INT 0x10
```
- **关键点**：
  - 使用BIOS中断 `INT 0x10` 与VBE规范交互。
  - 失败时回退到兼容性更强的VGA模式（模式0x13）。

---

#### **(2) 硬件初始化**
```nasm
keystatus:
; 获取键盘LED状态（通过BIOS）
MOV AH, 0x02
INT 0x16
MOV [LEDS], AL       ; 保存LED状态到内存

; 禁用PIC中断控制器
MOV AL, 0xff
OUT 0x21, AL         ; 屏蔽主PIC所有中断
OUT 0xa1, AL         ; 屏蔽从PIC所有中断
CLI                  ; 关闭CPU中断

; 启用A20地址线（访问1MB以上内存）
CALL waitkbdout
MOV AL, 0xd1
OUT 0x64, AL         ; 发送命令到键盘控制器
CALL waitkbdout
MOV AL, 0xdf         ; 启用A20
OUT 0x60, AL
```
- **关键点**：
  - **A20地址线**：突破8086的1MB内存限制，允许访问更高地址。
  - **PIC初始化**：防止硬件中断干扰引导过程。

---

#### **(3) 切换到保护模式**
```nasm
; 加载GDT并设置CR0.PE标志位
LGDT [GDTR0]         ; 加载GDT寄存器
MOV EAX, CR0
AND EAX, 0x7fffffff  ; 禁用分页（CR0.PG=0）
OR EAX, 0x00000001   ; 启用保护模式（CR0.PE=1）
MOV CR0, EAX
JMP pipelineflush    ; 清空指令流水线

pipelineflush:
; 设置段寄存器为保护模式下的数据段选择子
MOV AX, 1*8          ; GDT第1项：数据段（基址0，界限4GB）
MOV DS, AX
MOV ES, AX
MOV FS, AX
MOV GS, AX
MOV SS, AX
```
- **关键点**：
  - **GDT结构**：定义了代码段和数据段（见代码末尾的`GDT0`）。
  - **段选择子**：`1*8`对应GDT中的第二个描述符（索引1，TI=0，RPL=0）。

---

#### **(4) 加载内核到内存**
```nasm
; 复制bootpack（内核）到0x00280000（BOTPAK）
MOV ESI, bootpack    ; 源地址（当前代码中的bootpack标签）
MOV EDI, BOTPAK      ; 目标地址（0x00280000）
MOV ECX, 512*1024/4 ; 复制512KB（按4字节对齐）
CALL memcpy

; 复制磁盘数据到缓存区（0x00100000）
MOV ESI, 0x7c00      ; 引导扇区自身
MOV EDI, DSKCAC      ; 目标地址（0x00100000）
MOV ECX, 512/4       ; 复制512字节
CALL memcpy
```
- **关键点**：
  - `memcpy`函数按4字节块复制数据，提升效率。
  - `bootpack`是内核代码的起始标签，需在链接时确定位置。

---

#### **(5) 跳转到内核**
```nasm
; 设置内核栈指针并跳转
MOV ESP, [EBX+12]    ; 从bootpack头部获取栈指针
JMP DWORD 2*8:0x0000001b ; 远跳转：段选择子=2*8（代码段），偏移=0x1b
```
- **关键点**：
  - **段选择子**：`2*8`对应GDT中的第三个描述符（代码段）。
  - **内核入口**：偏移地址`0x1b`需与内核代码的入口点对齐。

---

### **3. 关键数据结构**
#### **(1) 全局描述符表（GDT）**
```nasm
GDT0:
    ; 空描述符（必须存在）
    DW 0, 0, 0, 0

    ; 数据段（基址0，界限4GB，可读/写）
    DW 0xffff      ; 段界限低16位
    DW 0x0000      ; 段基址低16位
    DB 0x00        ; 段基址中间8位
    DB 0x92        ; 访问权限：P=1, DPL=0, 数据段, 可写
    DB 0xcf        ; 标志位：G=1（4KB粒度）, D/B=1（32位）, AVL=0
    DB 0x00        ; 段基址高8位

    ; 代码段（基址0，界限4GB，可执行/读）
    DW 0xffff      ; 段界限低16位
    DW 0x0000      ; 段基址低16位
    DB 0x00        ; 段基址中间8位
    DB 0x9a        ; 访问权限：P=1, DPL=0, 代码段, 可读/执行
    DB 0xcf        ; 标志位：同上
    DB 0x00        ; 段基址高8位
```
- **作用**：定义保护模式下的内存段属性，供CPU进行地址转换和权限检查。

---

### **4. 关键函数**
#### **(1) `memcpy`**
```nasm
memcpy:
    MOV EAX, [ESI]    ; 读取4字节
    ADD ESI, 4
    MOV [EDI], EAX    ; 写入4字节
    ADD EDI, 4
    SUB ECX, 1
    JNZ memcpy
    RET
```
- **功能**：高效复制内存块（按4字节操作，减少循环次数）。

---

### **5. 总结**
`asmhead.nas` 是引导过程的 **第二阶段**，核心任务是：
1. **硬件初始化**（显示、中断、A20）。
2. **切换到保护模式**，突破实模式限制。
3. **加载内核到高内存地址**（如 `0x00280000`）。
4. **移交控制权**给内核，完成引导。

其设计体现了从 **实模式** 到 **保护模式** 的过渡逻辑，是操作系统启动过程中承上启下的关键环节。