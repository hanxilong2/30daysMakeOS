以下是 **`asmhead.nas` 的流程图与时序图**，基于其核心逻辑设计：

---

### **1. 流程图**
```mermaid
%% asmhead.nas 核心流程
graph TD
    A[开始执行] --> B[初始化显示模式]
    B -->|尝试VBE模式| C{是否支持VBE?}
    C -->|是| D[设置VBE模式0x105]
    C -->|否| E[回退到VGA模式0x13]
    D --> F[保存显示参数]
    E --> F
    F --> G[保存键盘LED状态]
    G --> H[禁用PIC中断]
    H --> I[启用A20地址线]
    I --> J[加载GDT]
    J --> K[设置CR0.PE=1进入保护模式]
    K --> L[更新段寄存器]
    L --> M[复制bootpack到内存]
    M --> N[复制磁盘缓存数据]
    N --> O[设置内核栈指针]
    O --> P[跳转执行内核]
    
    %% 子流程
    subgraph 显示模式初始化
        B --> C
        C --> D
        C --> E
    end
    
    subgraph 保护模式切换
        J --> K --> L
    end
```

---

### **2. 时序图**
```mermaid
%% asmhead.nas 时序交互
sequenceDiagram
    participant BIOS
    participant CPU
    participant 内存
    participant 键盘控制器
    participant 显示控制器
    
    BIOS ->> CPU: 执行asmhead.nas (0xc200)
    
    %% 显示模式初始化
    CPU ->> 显示控制器: INT 0x10/AX=0x4F00 (VBE信息)
    显示控制器 -->> CPU: 返回VBE支持状态
    alt VBE支持
        CPU ->> 显示控制器: INT 0x10/AX=0x4F02 (设置模式0x105)
        显示控制器 -->> CPU: 确认成功
    else VBE不支持
        CPU ->> 显示控制器: INT 0x10/AX=0x0013 (设置VGA模式)
    end
    
    %% 硬件初始化
    CPU ->> BIOS: INT 0x16/AH=0x02 (读取LED状态)
    BIOS -->> CPU: 返回LED状态
    CPU ->> 内存: 保存LED状态到0x0FF1
    
    CPU ->> 键盘控制器: OUT 0x64/0xD1 (A20命令)
    键盘控制器 -->> CPU: 等待响应
    CPU ->> 键盘控制器: OUT 0x60/0xDF (启用A20)
    
    %% 保护模式切换
    CPU ->> 内存: 加载GDT描述符
    CPU ->> CPU: LGDT [GDTR0]
    CPU ->> CPU: 设置CR0.PE=1
    CPU ->> CPU: JMP刷新流水线
    
    %% 数据复制与跳转
    CPU ->> 内存: 复制bootpack到0x00280000
    CPU ->> 内存: 复制磁盘数据到0x00100000
    CPU ->> CPU: 设置ESP为内核栈顶
    CPU ->> CPU: JMP 0x28:0x0000001B (内核入口)
```

---

### **关键步骤说明**
1. **显示模式初始化**  
   - 尝试使用VBE设置高分辨率模式（1024x768x8bit），失败则回退到VGA模式（320x200x8bit）。

2. **硬件状态保存与配置**  
   - 通过BIOS获取键盘LED状态（用于后续恢复）。  
   - 禁用PIC中断控制器，防止外部中断干扰。  
   - 通过键盘控制器启用A20地址线，解除1MB内存限制。

3. **保护模式切换**  
   - 加载GDT（全局描述符表），定义内存段的权限和属性。  
   - 设置CR0寄存器的PE位（Protection Enable），进入保护模式。  
   - 通过远跳转（Far Jump）更新段寄存器和指令流水线。

4. **内核加载与跳转**  
   - 使用`memcpy`函数将内核代码（`bootpack`）复制到高内存地址（`0x00280000`）。  
   - 复制磁盘缓存数据（如文件系统信息）到保留区域（`0x00100000`）。  
   - 设置内核栈指针，通过远跳转移交控制权到内核入口点。

---

### **核心理论背景**
1. **VBE显示模式**  
   - VBE（VESA BIOS Extensions）是BIOS的扩展规范，允许设置高分辨率显示模式。  
   - 模式`0x105`对应1024x768x8bit颜色，需验证颜色深度和内存布局是否匹配。

2. **保护模式切换**  
   - **GDT作用**：定义内存段的基址、界限和访问权限（如代码段/数据段）。  
   - **A20地址线**：突破8086的20位地址限制，使CPU可访问超过1MB的内存。

3. **内存复制优化**  
   - `memcpy`按4字节块操作，减少循环次数（`ECX = 总字节数/4`）。  
   - 使用`REP MOVSD`指令（本例未用）可进一步优化复制速度。

---

通过上述图表，可清晰理解 `asmhead.nas` 从硬件初始化到内核加载的完整逻辑链。