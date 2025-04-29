以下是在屏幕上绘制字符串 "Hello!" 的时序图，展示从初始化到最终显示的完整流程：

```mermaid
sequenceDiagram
    participant 应用层
    participant graphic.c
    participant 显存(vram)
    participant 字体数据(hankaku)

    应用层->>graphic.c: 1. 初始化调色板(init_palette)
    activate graphic.c
    graphic.c->>graphic.c: set_palette(0,15,预定义RGB)
    Note over graphic.c: 设置16色索引与实际RGB的映射
    deactivate graphic.c

    应用层->>graphic.c: 2. 获取屏幕信息(BOOTINFO)
    activate graphic.c
    graphic.c-->>应用层: 返回vram地址、分辨率等
    deactivate graphic.c

    应用层->>graphic.c: 3. 绘制背景(可选)
    activate graphic.c
    graphic.c->>显存(vram): boxfill8(vram,颜色,区域)
    deactivate graphic.c

    应用层->>graphic.c: 4. 绘制字符串"Hello!" (putfonts8_asc)
    activate graphic.c
    loop 遍历每个字符
        graphic.c->>字体数据(hankaku): 获取字符数据(如'h'=0x68)
        graphic.c->>显存(vram): putfont8(vram,x,y,颜色,字体数据)
        Note over graphic.c: x坐标增加8像素
    end
    deactivate graphic.c

    应用层->>硬件: 5. 显存刷新
    activate 硬件
    Note over 硬件: 显卡自动扫描vram输出到屏幕
    deactivate 硬件
```

### 流程说明：
1. **调色板初始化**  
   - 调用 `init_palette()` 设置16色模式的颜色值，建立颜色索引（如 `COL8_FFFFFF`）与物理RGB值的映射关系。

2. **获取显存信息**  
   - 通过 `BOOTINFO` 结构体获取显存基地址 `vram` 和屏幕分辨率（`scrnx`, `scrny`）。

3. **背景绘制（可选）**  
   - 使用 `boxfill8(vram, color, x0, y0, x1, y1)` 填充背景色（如清屏操作）。

4. **字符串绘制**  
   - 调用 `putfonts8_asc(vram, xsize, x, y, color, "Hello!")`：
     1. **遍历字符**：依次处理 'H'、'e'、'l'、'l'、'o'、'!'。
     2. **字体数据定位**：  
        - 字符 'H' 的 ASCII 码为 `0x48`，字体数据地址为 `hankaku + 0x48 * 16`。
     3. **逐像素绘制**：  
        - `putfont8` 将每个字符的 8x16 位图写入显存对应位置。
     4. **位置更新**：每绘制一个字符，X 坐标增加8像素（字符宽度）。

5. **显存刷新**  
   - 显卡硬件自动周期性地从 `vram` 读取像素数据并输出到屏幕，无需软件主动刷新。

---

### 关键函数协作图
```mermaid
graph TD
    A[主程序] --> B[init_palette]
    A --> C[获取BOOTINFO]
    A --> D[putfonts8_asc]
    D --> E{遍历字符}
    E --> F[计算字体地址 hankaku + ASCII*16]
    E --> G[调用putfont8绘制]
    G --> H["写入vram[y*scrnx + x]"]
    H --> I[更新x坐标]
    I --> E
```

### 技术细节
- **显存计算**：像素位置 `(x,y)` 对应显存地址为 `vram[y * scrnx + x]`。
- **字体编码**：每个字符的字体数据为16字节，每字节表示一行8个像素（高位在左）。
- **颜色替换**：`putfont8` 将字体数据中的有效位（1）替换为指定颜色值，透明位（0）保留背景色。