以下是该项目的启动流程详细解读（Makefile文件）及对应的Mermaid流程图：

```mermaid
graph TD
    A[开始] --> B[编译引导程序]
    B --> B1[ipl10.nas]
    B1 --> B2(nask编译)
    B2 --> B3[ipl10.bin]
    
    A --> C[编译内核头部]
    C --> C1[asmhead.nas]
    C1 --> C2(nask编译)
    C2 --> C3[asmhead.bin]
    
    A --> D[生成字体数据]
    D --> D1[hankaku.txt]
    D1 --> D2(makefont转换)
    D2 --> D3[hankaku.bin]
    D3 --> D4(bin2obj转换)
    D4 --> D5[hankaku.obj]
    
    A --> E[编译内核组件]
    E --> E1[*.c文件]
    E1 --> E2(CC1预处理)
    E2 --> E3[*.gas文件]
    E3 --> E4(gas2nask转换)
    E4 --> E5[*.nas文件]
    E5 --> E6(nask编译)
    E6 --> E7[多个.obj文件]
    
    E7 --> F[链接内核]
    F --> F1(obj2bim链接)
    F1 --> F2[bootpack.bim]
    F2 --> F3(bim2hrb转换)
    F3 --> F4[bootpack.hrb]
    
    C3 & F4 --> G[合并内核]
    G --> G1(二进制合并)
    G1 --> G2[haribote.sys]
    
    B3 & G2 --> H[创建磁盘镜像]
    H --> H1(edimg打包)
    H1 --> H2[haribote.img]
    
    H2 --> I[运行QEMU]
    I --> I1[复制镜像]
    I1 --> I2[启动模拟器]
```

### 流程说明：
1. **引导程序编译阶段**
   - 使用nask汇编器编译ipl10.nas生成512字节引导扇区文件ipl10.bin

2. **内核头部编译**
   - 编译asmhead.nas生成内核头部二进制asmhead.bin

3. **字体数据生成**
   - 将hankaku.txt字库文件转换为二进制格式
   - 将二进制字体数据转换为目标文件格式

4. **内核组件编译**
   - 使用定制工具链（CC1+GAS2NASK）编译C源文件
   - 分阶段处理：C代码→gas中间格式→nasm汇编→目标文件

5. **内核链接与转换**
   - 使用obj2bim链接所有目标文件生成bim中间文件
   - 通过bim2hrb转换为可执行格式hrb文件

6. **系统镜像构建**
   - 合并内核头部和主程序（asmhead.bin + bootpack.hrb）
   - 使用edimg工具构建完整磁盘镜像：
     - 插入引导程序到0扇区
     - 写入内核文件到文件系统区域

7. **模拟运行**
   - 将生成的img镜像复制到QEMU目录
   - 启动虚拟机加载镜像运行

### 关键工具说明：
- **nask**: 专用汇编编译器
- **CC1+GAS2NASK**: 定制C编译工具链
- **obj2bim/bim2hrb**: 专用的链接转换工具
- **edimg**: 磁盘镜像构建工具
- **makefont**: 字库转换工具

这个构建系统展示了典型的操作系统开发流程，包含了从底层引导程序到高层内核组件的完整构建过程，最终生成可直接在虚拟机运行的磁盘镜像文件。