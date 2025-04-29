以下是 **sheet.c** 中各函数的详细功能介绍，结合关键代码片段解析其作用和实现逻辑：

---

### **1. `shtctl_init` - 初始化图层控制器**
```c
struct SHTCTL *shtctl_init(struct MEMMAN *memman, unsigned char *vram, int xsize, int ysize) {
    struct SHTCTL *ctl;
    ctl = memman_alloc_4k(memman, sizeof(struct SHTCTL)); // 分配控制器内存
    ctl->map = memman_alloc_4k(memman, xsize * ysize);    // 分配像素映射表
    ctl->vram = vram;          // 绑定显存
    ctl->xsize = xsize;        // 屏幕宽度
    ctl->ysize = ysize;        // 屏幕高度
    ctl->top = -1;             // 初始无可见图层
    for (int i = 0; i < MAX_SHEETS; i++) {
        ctl->sheets0[i].flags = 0; // 标记所有图层未使用
    }
    return ctl;
}
```
- **作用**：创建图层控制器，分配内存并初始化核心参数。
- **关键代码**：
  - `memman_alloc_4k`：从内存管理器分配4KB对齐的内存。
  - `ctl->top = -1`：初始时无可见图层。

---

### **2. `sheet_alloc` - 分配图层对象**
```c
struct SHEET *sheet_alloc(struct SHTCTL *ctl) {
    for (int i = 0; i < MAX_SHEETS; i++) {
        if (ctl->sheets0[i].flags == 0) {    // 查找未使用的图层
            struct SHEET *sht = &ctl->sheets0[i];
            sht->flags = SHEET_USE;          // 标记为已使用
            sht->height = -1;               // 初始隐藏
            return sht;
        }
    }
    return 0; // 无可用图层
}
```
- **作用**：从预分配池 `sheets0` 中获取一个空闲图层。
- **关键代码**：
  - `flags == 0`：检查图层是否空闲。
  - `height = -1`：新图层默认隐藏。

---

### **3. `sheet_setbuf` - 设置图层缓冲区**
```c
void sheet_setbuf(struct SHEET *sht, unsigned char *buf, int xsize, int ysize, int col_inv) {
    sht->buf = buf;         // 绑定像素缓冲区
    sht->bxsize = xsize;    // 逻辑宽度
    sht->bysize = ysize;    // 逻辑高度
    sht->col_inv = col_inv; // 透明色
}
```
- **作用**：绑定图层的像素数据及属性，为绘制做准备。

---

### **4. `sheet_refreshmap` - 更新像素归属映射**
```c
void sheet_refreshmap(struct SHTCTL *ctl, int vx0, int vy0, int vx1, int vy1, int h0) {
    for (int h = h0; h <= ctl->top; h++) {         // 遍历指定高度以上的图层
        struct SHEET *sht = ctl->sheets[h];
        unsigned char sid = sht - ctl->sheets0;    // 计算图层ID
        for (int by = by0; by < by1; by++) {
            for (int bx = bx0; bx < bx1; bx++) {
                if (sht->buf[by * sht->bxsize + bx] != sht->col_inv) {
                    ctl->map[vy * ctl->xsize + vx] = sid; // 记录非透明像素的图层ID
                }
            }
        }
    }
}
```
- **作用**：更新屏幕区域的映射表，标记每个像素由哪个图层占据。
- **关键代码**：
  - `sid = sht - ctl->sheets0`：通过指针差值计算唯一图层ID。
  - 仅处理非透明像素（`!= col_inv`）。

---

### **5. `sheet_refreshsub` - 局部屏幕刷新**
```c
void sheet_refreshsub(...) {
    for (int h = h0; h <= h1; h++) {         // 遍历指定高度范围的图层
        struct SHEET *sht = ctl->sheets[h];
        for (int by = by0; by < by1; by++) {
            for (int bx = bx0; bx < bx1; bx++) {
                if (ctl->map[vy * ctl->xsize + vx] == sid) { // 检查是否为当前图层
                    ctl->vram[vy * ctl->xsize + vx] = sht->buf[by * sht->bxsize + bx];
                }
            }
        }
    }
}
```
- **作用**：根据映射表 `map` 将可见像素绘制到显存。
- **关键代码**：
  - `map[位置] == sid`：仅绘制当前图层占据的像素。

---

### **6. `sheet_updown` - 调整图层高度**
```c
void sheet_updown(struct SHEET *sht, int height) {
    // 调整高度范围限制
    if (height > ctl->top + 1) height = ctl->top + 1;
    if (height < -1) height = -1;

    int old = sht->height;
    sht->height = height;

    // 高度降低时的处理
    if (old > height) {
        for (int h = old; h > height; h--) {
            ctl->sheets[h] = ctl->sheets[h - 1]; // 上层图层下移
            ctl->sheets[h]->height = h;
        }
        ctl->sheets[height] = sht; // 插入当前图层
    }
    // 触发局部刷新
    sheet_refreshmap(ctl, sht->vx0, sht->vy0, sht->vx0 + sht->bxsize, sht->vy0 + sht->bysize, height + 1);
    sheet_refreshsub(ctl, sht->vx0, sht->vy0, sht->vx0 + sht->bxsize, sht->vy0 + sht->bysize, height + 1, old);
}
```
- **作用**：调整图层的显示优先级（Z轴顺序），并触发刷新。
- **关键代码**：
  - `sheets[h] = sheets[h-1]`：通过数组移位实现图层顺序调整。
  - 仅刷新受影响区域（新旧高度之间的图层）。

---

### **7. `sheet_slide` - 移动图层位置**
```c
void sheet_slide(struct SHEET *sht, int vx0, int vy0) {
    int old_vx0 = sht->vx0, old_vy0 = sht->vy0;
    sht->vx0 = vx0; // 新X坐标
    sht->vy0 = vy0; // 新Y坐标
    if (sht->height >= 0) { // 若图层可见
        // 刷新旧位置和新位置的映射及显存
        sheet_refreshmap(ctl, old_vx0, old_vy0, old_vx0 + sht->bxsize, old_vy0 + sht->bysize, 0);
        sheet_refreshmap(ctl, vx0, vy0, vx0 + sht->bxsize, vy0 + sht->bysize, sht->height);
        sheet_refreshsub(ctl, old_vx0, old_vy0, old_vx0 + sht->bxsize, old_vy0 + sht->bysize, 0, sht->height - 1);
        sheet_refreshsub(ctl, vx0, vy0, vx0 + sht->bxsize, vy0 + sht->bysize, sht->height, sht->height);
    }
}
```
- **作用**：移动图层并刷新新旧区域，避免残影。
- **关键代码**：
  - 同时处理旧位置（擦除）和新位置（绘制）。

---

### **8. `sheet_free` - 释放图层资源**
```c
void sheet_free(struct SHEET *sht) {
    if (sht->height >= 0) {
        sheet_updown(sht, -1); // 先隐藏图层
    }
    sht->flags = 0; // 标记为未使用
}
```
- **作用**：释放图层，归还到控制器池中。

---

### **关键设计亮点**
1. **透明色优化**  
   通过 `col_inv` 跳过无效像素绘制，减少计算量：
   ```c
   if (buf[by * sht->bxsize + bx] != sht->col_inv) { ... }
   ```

2. **局部刷新机制**  
   仅更新受影响的屏幕区域（如移动鼠标时刷新新旧位置）：
   ```c
   sheet_refreshsub(ctl, old_vx0, old_vy0, ..., 0, sht->height - 1);
   sheet_refreshsub(ctl, new_vx0, new_vy0, ..., sht->height, sht->height);
   ```

3. **高度排序算法**  
   通过数组移位动态维护图层顺序，时间复杂度为 O(n)：
   ```c
   for (int h = old; h > height; h--) {
       ctl->sheets[h] = ctl->sheets[h - 1];
   }
   ```

