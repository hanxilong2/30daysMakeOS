以下是 `mtask.c` 中各个函数的详细功能解析，结合代码逻辑和多任务系统的核心机制：

---

### **1. `struct TASK *task_now(void)`**
- **功能**：获取当前正在运行的任务。
- **实现逻辑**：
  - 通过 `taskctl->now_lv` 获取当前活跃的任务层级。
  - 从该层级的任务队列 (`tl->tasks`) 中，根据 `tl->now` 索引返回当前任务。
- **关键代码**：
  ```c
  struct TASKLEVEL *tl = &taskctl->level[taskctl->now_lv];
  return tl->tasks[tl->now];
  ```
- **作用**：用于确定当前 CPU 正在执行的任务，常用于任务休眠或优先级调整。

---

### **2. `void task_add(struct TASK *task)`**
- **功能**：将任务添加到指定优先级的运行队列。
- **参数**：`task` 需添加到队列的任务。
- **实现逻辑**：
  - 根据任务的 `level` 属性找到对应的优先级队列。
  - 将任务添加到队列末尾，并更新队列的 `running` 计数。
  - 设置任务的 `flags` 为 `2`（运行状态）。
- **关键代码**：
  ```c
  struct TASKLEVEL *tl = &taskctl->level[task->level];
  tl->tasks[tl->running] = task;
  tl->running++;
  task->flags = 2;
  ```
- **作用**：管理任务的就绪状态，使其可被调度器选择。

---

### **3. `void task_remove(struct TASK *task)`**
- **功能**：从运行队列中移除任务。
- **参数**：`task` 需移除的任务。
- **实现逻辑**：
  1. 遍历当前层级的任务队列，找到任务的位置。
  2. 调整队列的 `running` 计数和 `now` 指针（避免索引越界）。
  3. 将后续任务前移，覆盖被移除的任务。
  4. 设置任务的 `flags` 为 `1`（休眠状态）。
- **关键代码**：
  ```c
  for (i = 0; i < tl->running; i++) {
      if (tl->tasks[i] == task) break;
  }
  // ...调整队列并收缩...
  task->flags = 1;
  ```
- **作用**：用于任务休眠或结束时，将其移出调度队列。

---

### **4. `void task_switchsub(void)`**
- **功能**：切换当前活跃的任务层级（优先级）。
- **实现逻辑**：
  - 遍历所有优先级层级，找到第一个非空的队列。
  - 更新 `taskctl->now_lv` 为新的层级。
  - 重置层级切换标志 `lv_change`。
- **关键代码**：
  ```c
  for (i = 0; i < MAX_TASKLEVELS; i++) {
      if (taskctl->level[i].running > 0) break;
  }
  taskctl->now_lv = i;
  ```
- **作用**：确保调度器始终运行最高优先级的就绪任务。

---

### **5. `void task_idle(void)`**
- **功能**：空闲任务，在没有其他任务运行时执行。
- **实现逻辑**：
  - 无限循环调用 `io_hlt()` 进入低功耗状态。
  ```c
  for (;;) io_hlt();
  ```
- **作用**：节省 CPU 资源，避免忙等待。

  - 处于最低优先级队列，是为了解决所有任务都处于休眠状态时而存在的。


---

### **6. `struct TASK *task_init(struct MEMMAN *memman)`**
- **功能**：初始化任务管理系统，创建初始任务和空闲任务。
- **实现逻辑**：
  1. 分配内存给任务控制器 `taskctl`。
  2. 初始化 GDT 中的 TSS 描述符，每个任务占用一个描述符。
  3. 创建初始任务 `task`（任务A）并设置其优先级和层级。
  4. 创建空闲任务 `idle`，绑定到最低优先级。
- **关键代码**：
  ```c
  taskctl = memman_alloc_4k(memman, sizeof(struct TASKCTL));
  set_segmdesc(gdt + TASK_GDT0 + i, 103, &taskctl->tasks0[i].tss, AR_TSS32);
  task_run(idle, MAX_TASKLEVELS - 1, 1); // 空闲任务在最低层级
  ```
- **作用**：构建多任务环境的基础设施。

---

### **7. `struct TASK *task_alloc(void)`**
- **功能**：从任务池中分配一个空闲任务。
- **实现逻辑**：
  - 遍历 `taskctl->tasks0` 数组，找到第一个 `flags == 0` 的任务。
  - 初始化任务的 TSS 字段（寄存器清零、I/O 权限位图等）。
  - 设置 `flags = 1`（标记为已分配但未运行）。
- **关键代码**：
  ```c
  task->tss.eflags = 0x00000202; // IF=1（允许中断）
  task->tss.iomap = 0x40000000;  // 禁用I/O权限检查
  ```
- **作用**：动态创建新任务，如窗口管理或后台进程。

---

### **8. `void task_run(struct TASK *task, int level, int priority)`**
- **功能**：启动或重新调度任务，设置其优先级。
- **参数**：
  - `level`：目标优先级层级（若为负则保持当前层级）。
  - `priority`：时间片长度（单位为定时器中断周期）。
- **实现逻辑**：
  - 若任务已在运行且需切换层级，先调用 `task_remove`。
  - 将任务添加到新层级的队列，并标记层级需要切换。
- **关键代码**：
  ```c
  if (task->flags == 2 && task->level != level) {
      task_remove(task);
  }
  task_add(task); // 添加到新队列
  taskctl->lv_change = 1; // 触发层级切换
  ```
- **作用**：动态调整任务优先级或唤醒休眠任务。

---

### **9. `void task_sleep(struct TASK *task)`**
- **功能**：使任务进入休眠状态。
- **实现逻辑**：
  1. 从运行队列中移除任务。
  2. 若休眠的是当前任务，立即触发任务切换。
  3. 调用 `task_switchsub` 更新活跃层级。
  4. 通过 `farjmp` 跳转到新任务的 TSS。
- **关键代码**：
  ```c
  if (task == now_task) {
      task_switchsub();
      farjmp(0, now_task->sel); // 强制切换
  }
  ```
- **作用**：处理任务等待事件（如无输入时休眠）。

---

### **10. `void task_switch(void)`**
- **功能**：执行实际的任务切换操作。
- **实现逻辑**：
  1. 从当前层级的队列中选择下一个任务（轮转调度）。
  2. 检查是否需要切换层级（`lv_change` 标志）。
  3. 重置定时器时间为新任务的时间片。
  4. 通过 `farjmp` 切换到新任务的 TSS。
- **关键代码**：
  ```c
  tl->now = (tl->now + 1) % tl->running; // 轮转
  farjmp(0, new_task->sel); // CPU加载新TSS
  ```
- **作用**：实现时间片轮转调度和优先级切换。

---

### **总结**
- **任务生命周期管理**：通过 `task_alloc` 和 `task_remove` 控制任务的创建和销毁。
- **调度策略**：结合优先级层级 (`TASKLEVEL`) 和时间片轮转 (`task_switch`)，实现多级反馈队列调度。
- **中断协同**：定时器中断 (`inthandler20`) 触发调度，任务休眠和唤醒通过 FIFO 事件机制联动。

通过以上函数，系统能够高效管理多个任务的执行、优先级调整和资源分配。