以下是 `memory.c` 中各函数的详细解析，涵盖了它们的核心作用、实现逻辑和关键代码片段：

---

### 1. **`memtest`：物理内存检测**
```c
unsigned int memtest(unsigned int start, unsigned int end) {
    // 检测CPU是否为486及以上（通过设置EFLAGS的AC位）
    eflg = io_load_eflags();
    eflg |= EFLAGS_AC_BIT; 
    io_store_eflags(eflg);
    if ((eflg & EFLAGS_AC_BIT) != 0) flg486 = 1;

    // 禁用缓存（保证直接读写物理内存）
    if (flg486) {
        cr0 = load_cr0();
        cr0 |= CR0_CACHE_DISABLE; 
        store_cr0(cr0);
    }

    // 调用汇编实现的memtest_sub进行实际测试
    i = memtest_sub(start, end);

    // 恢复缓存设置
    if (flg486) {
        cr0 &= ~CR0_CACHE_DISABLE;
        store_cr0(cr0);
    }
    return i; // 返回最大可用内存地址
}
```
- **作用**：检测从 `start` 到 `end` 的物理内存是否可用。
- **关键逻辑**：
  - 通过修改 `CR0` 寄存器**禁用CPU缓存**，确保直接访问物理内存。
  - 调用底层汇编函数 `memtest_sub` 进行模式测试（如写入 `0xAA55AA55` 并验证）。
  - 返回最后一个可用的内存地址（若遇到坏内存则提前终止）。

---

### 2. **`memman_init`：初始化内存管理器**
```c
void memman_init(struct MEMMAN *man) {
    man->frees = 0;       // 初始空闲块数为0
    man->maxfrees = 0;    // 最大空闲块数初始化为0
    man->lostsize = 0;    // 丢失内存总量清零
    man->losts = 0;       // 丢失次数清零
    return;
}
```
- **作用**：初始化 `MEMMAN` 结构体，清零所有统计字段。
- **设计意图**：为内存管理器提供一个干净的初始状态。

---

### 3. **`memman_total`：计算总空闲内存**
```c
unsigned int memman_total(struct MEMMAN *man) {
    unsigned int t = 0;
    for (int i = 0; i < man->frees; i++) {
        t += man->free[i].size; // 累加所有空闲块大小
    }
    return t; // 返回总空闲内存字节数
}
```
- **作用**：遍历空闲块数组，计算当前可用内存总量。
- **时间复杂度**：O(n)，n为当前空闲块数。

---

### 4. **`memman_alloc`：内存分配（核心）**
```c
unsigned int memman_alloc(struct MEMMAN *man, unsigned int size) {
    for (int i = 0; i < man->frees; i++) {
        if (man->free[i].size >= size) { // 首次适应算法
            unsigned int a = man->free[i].addr;
            man->free[i].addr += size;    // 调整起始地址
            man->free[i].size -= size;    // 减少块大小
            if (man->free[i].size == 0) { // 块完全分配后删除
                man->frees--;
                for (; i < man->frees; i++) {
                    man->free[i] = man->free[i+1]; // 前移后续块
                }
            }
            return a; // 返回分配地址
        }
    }
    return 0; // 分配失败返回0
}
```
- **作用**：分配指定大小的内存块。
- **关键逻辑**：
  - **首次适应算法**：找到第一个足够大的空闲块。
  - **分割机制**：若块大小大于需求，分割剩余部分留在数组中。
  - **完全分配处理**：若块大小正好匹配，从数组中移除该块。
- **失败条件**：无足够大的空闲块时返回0。

---

### 5. **`memman_free`：内存释放（核心）**
```c
int memman_free(struct MEMMAN *man, unsigned int addr, unsigned int size) {
    int i, j;
    // 查找插入位置（按地址排序）
    for (i = 0; i < man->frees; i++) {
        if (man->free[i].addr > addr) break;
    }
    
    // 前向合并
    if (i > 0 && man->free[i-1].addr + man->free[i-1].size == addr) {
        man->free[i-1].size += size;
        // 后向合并（如果可能）
        if (i < man->frees && addr + size == man->free[i].addr) {
            man->free[i-1].size += man->free[i].size;
            man->frees--;
            for (; i < man->frees; i++) man->free[i] = man->free[i+1];
        }
        return 0; // 合并成功
    }
    
    // 后向合并
    if (i < man->frees && addr + size == man->free[i].addr) {
        man->free[i].addr = addr;
        man->free[i].size += size;
        return 0; 
    }
    
    // 无法合并，插入新块
    if (man->frees < MEMMAN_FREES) {
        for (j = man->frees; j > i; j--) man->free[j] = man->free[j-1];
        man->frees++;
        man->free[i].addr = addr;
        man->free[i].size = size;
        if (man->maxfrees < man->frees) man->maxfrees = man->frees;
        return 0;
    }
    
    // 数组已满，记录丢失
    man->losts++;
    man->lostsize += size;
    return -1; // 释放失败
}
```
- **作用**：释放内存并尝试合并相邻块。
- **关键逻辑**：
  - **插入位置查找**：维护空闲块按地址升序排列。
  - **前向/后向合并**：检查释放块是否与前后块相邻，合并减少碎片。
  - **数组满处理**：若无法插入新块，统计丢失内存。
- **返回值**：成功返回0，失败返回-1。

---

### 6. **`memman_alloc_4k`：4K对齐分配**
```c
unsigned int memman_alloc_4k(struct MEMMAN *man, unsigned int size) {
    size = (size + 0xfff) & 0xfffff000; // 向上对齐到4K边界
    return memman_alloc(man, size);
}
```
- **作用**：分配4KB对齐的内存（适用于分页机制）。
- **对齐逻辑**：通过位运算将 `size` 调整为4KB的倍数（如 `size=4097` 对齐到 `8192`）。

---

### 7. **`memman_free_4k`：4K对齐释放**
```c
int memman_free_4k(struct MEMMAN *man, unsigned int addr, unsigned int size) {
    size = (size + 0xfff) & 0xfffff000; // 对齐到4K边界
    return memman_free(man, addr, size);
}
```
- **作用**：释放通过 `memman_alloc_4k` 分配的内存。
- **必要性**：确保释放的大小与分配时对齐的大小一致。

---

### 关键设计总结
| 函数名            | 核心功能       | 时间复杂度 | 关键操作             |
| ----------------- | -------------- | ---------- | -------------------- |
| `memtest`         | 物理内存检测   | O(n)       | 禁用缓存、模式测试   |
| `memman_init`     | 初始化管理器   | O(1)       | 清零统计字段         |
| `memman_total`    | 计算总空闲内存 | O(n)       | 遍历累加             |
| `memman_alloc`    | 分配内存       | O(n)       | 首次适应、分割块     |
| `memman_free`     | 释放内存       | O(n)       | 合并相邻块、维护数组 |
| `memman_alloc_4k` | 4K对齐分配     | O(n)       | 对齐计算、调用分配   |
| `memman_free_4k`  | 4K对齐释放     | O(n)       | 对齐计算、调用释放   |

- **碎片控制**：通过合并策略减少外部碎片，但无法解决内部碎片（如4K对齐必然产生的浪费）。
- **性能瓶颈**：线性搜索（O(n)）在空闲块数多时效率较低，适合小型嵌入式系统。
- **鲁棒性**：通过 `lostsize` 和 `losts` 监控内存管理异常，辅助调试。