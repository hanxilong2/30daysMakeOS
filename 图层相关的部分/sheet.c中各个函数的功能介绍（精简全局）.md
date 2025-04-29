以下是 **sheet.c** 中所有函数的功能解析，涵盖其作用、参数意义及关键实现逻辑：

---

### **1. `struct SHTCTL *shtctl_init(...)`**
```c
struct SHTCTL *shtctl_init(struct MEMMAN *memman, unsigned char *vram, int xsize, int ysize);
```
- **功能**：初始化图层控制器（SHTCTL），分配内存并设置初始状态。
- **参数**：
  - `memman`：内存管理器，用于动态分配内存。
  - `vram`：显存起始地址。
  - `xsize`, `ysize`：屏幕分辨率。
- **关键操作**：
  - 分配 `SHTCTL` 结构体内存。
  - 分配 `map` 映射表内存（记录每个像素的顶层图层）。
  - 初始化所有图层为未使用状态（`flags = 0`）。
- **返回值**：成功返回控制器指针，失败返回 `0`。

---

### **2. `struct SHEET *sheet_alloc(struct SHTCTL *ctl)`**
```c
struct SHEET *sheet_alloc(struct SHTCTL *ctl);
```
- **功能**：从控制器的预分配池 `sheets0` 中获取一个未使用的图层。
- **关键操作**：
  - 遍历 `sheets0` 数组，找到 `flags == 0` 的图层。
  - 标记为已使用（`flags = SHEET_USE`），并设置初始高度为 `-1`（隐藏）。
- **返回值**：成功返回图层指针，失败返回 `0`（无可用图层）。

---

### **3. `void sheet_setbuf(...)`**
```c
void sheet_setbuf(struct SHEET *sht, unsigned char *buf, int xsize, int ysize, int col_inv);
```
- **功能**：设置图层的像素缓冲区及属性。
- **参数**：
  - `sht`：目标图层。
  - `buf`：像素缓冲区指针。
  - `xsize`, `ysize`：图层的逻辑尺寸。
  - `col_inv`：透明色索引（该颜色不会被绘制）。
- **作用**：绑定图层与缓冲区，为后续绘制和刷新做准备。

---

### **4. `void sheet_refreshmap(...)`**
```c
void sheet_refreshmap(struct SHTCTL *ctl, int vx0, int vy0, int vx1, int vy1, int h0);
```
- **功能**：更新屏幕区域 `[vx0, vy0]` 到 `[vx1, vy1]` 的图层映射表（`map`）。
- **参数**：
  - `h0`：从该高度以上的图层开始更新。
- **关键逻辑**：
  - 遍历从 `h0` 到顶层 `top` 的所有图层。
  - 对于每个像素，若颜色非透明（`buf[by][bx] != col_inv`），则在 `map` 中记录该图层 ID。
- **作用**：确定每个像素当前由哪个图层占据，为绘制优化提供依据。

---

### **5. `void sheet_refreshsub(...)`**
```c
void sheet_refreshsub(struct SHTCTL *ctl, int vx0, int vy0, int vx1, int vy1, int h0, int h1);
```
- **功能**：根据映射表 `map` 重新绘制屏幕区域 `[vx0, vy0]` 到 `[vx1, vy1]`。
- **参数**：
  - `h0`, `h1`：仅处理高度在 `h0` 到 `h1` 之间的图层。
- **关键逻辑**：
  - 遍历指定高度范围内的所有图层。
  - 对于每个像素，若 `map` 中记录的图层 ID 与当前图层匹配，则将缓冲区像素复制到显存（`vram`）。
- **作用**：局部刷新屏幕，避免全屏重绘。

---

### **6. `void sheet_updown(struct SHEET *sht, int height)`**
```c
void sheet_updown(struct SHEET *sht, int height);
```
- **功能**：调整图层的高度（显示优先级）。
- **参数**：
  - `height`：目标高度（`-1` 表示隐藏，数值越大越靠前）。
- **关键操作**：
  - **隐藏图层**：若 `height = -1`，将其从 `sheets[]` 数组中移除，并更新 `top`。
  - **提升高度**：若 `height > 原高度`，将上层图层下移，插入当前图层。
  - **降低高度**：若 `height < 原高度`，将下层图层上移，插入当前图层。
  - 触发 `sheet_refreshmap` 和 `sheet_refreshsub` 刷新受影响区域。

---

### **7. `void sheet_refresh(struct SHEET *sht, ...)`**
```c
void sheet_refresh(struct SHEET *sht, int bx0, int by0, int bx1, int by1);
```
- **功能**：局部刷新图层的指定区域（缓冲区坐标）。
- **参数**：
  - `bx0`, `by0`：缓冲区区域的左上角。
  - `bx1`, `by1`：缓冲区区域的右下角。
- **关键逻辑**：
  - 将缓冲区坐标转换为屏幕坐标（`vx0 + bx0` 等）。
  - 调用 `sheet_refreshsub` 仅刷新该区域。

---

### **8. `void sheet_slide(struct SHEET *sht, int vx0, int vy0)`**
```c
void sheet_slide(struct SHEET *sht, int vx0, int vy0);
```
- **功能**：移动图层到新的屏幕位置 `(vx0, vy0)`。
- **关键操作**：
  - 记录旧位置，更新新位置。
  - 若图层可见（`height >= 0`），刷新旧位置和新位置的区域：
    - `sheet_refreshmap` 更新新旧区域的映射表。
    - `sheet_refreshsub` 重绘新旧区域。
- **典型应用**：移动鼠标光标时调用。

---

### **9. `void sheet_free(struct SHEET *sht)`**
```c
void sheet_free(struct SHEET *sht);
```
- **功能**：释放图层资源，将其标记为未使用。
- **关键操作**：
  - 若图层可见（`height >= 0`），先调用 `sheet_updown` 隐藏。
  - 重置 `flags = 0`，归还到控制器池中。

---

### **关键协作流程**
1. **图层移动**  
   `sheet_slide` → 更新位置 → 刷新旧位置和新位置。  
2. **高度调整**  
   `sheet_updown` → 调整 `sheets[]` 数组 → 刷新受影响区域。  
3. **局部刷新**  
   `sheet_refresh` → 转换为屏幕坐标 → 调用 `sheet_refreshsub`。  

### **性能优化**
- **映射表 (`map`)**：避免逐层遍历所有图层，直接通过 `map` 确定顶层像素。
- **局部刷新**：仅处理实际变化的屏幕区域，减少计算量。

