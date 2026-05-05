+++
draft = false
date = 2026-05-05T22:00:00+08:00
title = "从零开始写一个终端模拟器：基于 libghostty-vt 的极简实现与进阶之路"
description = ""
slug = ""
authors = []
tags = ["终端", "libghostty-vt", "Qt", "技术"]
categories = []
externalLink = ""
series = []

+++

为什么基于 libghotty-vt 做新版的 deepin-terminal，在上篇博客中已经介绍过了。

这个项目做下来（还未完成），最大的感受是：终端这玩意看着简单，底层知识缺口一个接一个。终端怎么跟 shell 通信、shell 集成到底集成的是不是 shell、各种测试工具怎么搭……这些问题之前完全没概念。这篇文章就是一路填坑的笔记，从概念到代码，说说写一个终端模拟器到底要搞清哪些东西。

---



要写终端，Shell、PTY、Terminal Emulator、VT 序列这四个东西的关系必须搞明白。我一开始也是糊里糊涂的，以为 Terminal Emulator 就是"一个带图形界面的 shell"，其实不是。

### 1.1 Shell

Shell 就是 bash、zsh、fish 这些东西。它从你给它的标准输入读命令，解析完 fork 个子进程去跑，再把结果写回标准输出。Shell 根本不在乎输出到了哪里，管道也好，文件也好，PTY 也好，对它来说都是文件描述符。

### 1.2 PTY

终端模拟器是 GUI 程序，用户通过窗口和它交互。但 Shell 和它启动的命令程序，骨子里仍觉得自己应该连在一个字符设备上。这个设备叫 TTY（电传打字机的历史遗留），程序通过检测 stdout 是不是 TTY 来决定要不要输出彩色文本、显示进度条、启动 TUI 界面。如果直接把 Shell 的标准输出接到管道上，它立刻进入哑巴模式，输出变成灰扑扑的纯文本。

PTY（Pseudo Terminal，伪终端）就是来解决这个矛盾的。内核提供了一对虚拟设备文件：master 和 slave。Shell 和命令程序连在 slave 端，把它们的标准输入输出都绑到这里。slave 端的行为跟真实 TTY 一模一样，所以程序检测 stdout 时会得到"是 TTY"的结果，正常输出颜色和转义序列。终端模拟器则连在 master 端，读取程序输出、发送用户输入。

Linux 上 `forkpty()` 这个调用一次搞定：创建 PTY 对、fork 子进程、在子进程里把 slave 绑到标准 IO、在父进程返回 master fd。不用自己手动 `openpty` 再搞一堆 ioctl。

### 1.3 Terminal Emulator

终端模拟器就是你要写的 GUI 程序。它的活儿就这几件：

- 从 PTY master 读程序输出
- 解析 VT 序列，搞清楚"光标移到第 3 行第 5 列""这行字变红色"是什么意思
- 把字符画到屏幕上
- 把你敲的键盘、点的鼠标编码成字节流，发回 PTY master

翻译器。一边是人类的眼睛和手，另一边是程序能读的字节流。

### 1.4 VT 序列

VT 序列也叫转义序列，终端和程序之间的暗号。常见的几个：

- `\x1b[2J` 清屏
- `\x1b[31m` 红色前景
- `\x1b[H` 光标回左上角
- `\x1b]0;标题\x07` 设置窗口标题

这堆东西的解析和状态维护非常烦人。状态机、参数解析、嵌套逻辑、各家厂商的扩展，自己从头写的话光兼容 xterm 就能磨掉几个月。

libghostty-vt 干的就是这个脏活。

### 1.5 数据流

整条链路其实不复杂：

用户打字 → Terminal Emulator 编码成 VT 序列 → 写进 PTY master → 内核转到 PTY slave → Shell 读到 → 执行命令 → 命令输出文本和 VT 序列 → 写进 PTY slave → 内核转回 PTY master → Terminal Emulator 读到 → 丢给 libghostty-vt 解析 → 更新状态 → 重绘屏幕

理解了这条线，写终端就剩挨个填坑了。

---

## 二、libghostty-vt 

libghostty-vt 是从 Ghostty 终端抽出来的 C 库，负责终端模拟器里最累最脏的部分：

- 解析 VT 转义序列
- 维护终端状态（屏幕缓冲区、光标位置、滚动历史、样式属性）
- 提供增量渲染接口
- 键盘和鼠标事件的编码（Kitty Keyboard Protocol、SGR Mouse）

UI、网络、PTY 它一概不碰，全留给宿主程序。这种设计很干净，Qt、GTK、甚至自己写窗口系统都能包一层。

API 围绕几个 handle 展开：

- `GhosttyTerminal`：终端状态机本体
- `GhosttyRenderState`：渲染快照，带脏跟踪
- `GhosttyKeyEncoder` / `GhosttyKeyEvent`：键盘编码
- `GhosttyMouseEncoder` / `GhosttyMouseEvent`：鼠标编码

创建和销毁都要检查 `GhosttyResult`，释放顺序必须跟创建顺序反着来，这点后面会说到。

---

## 三、动手：搭一个能跑 shell 的最小终端

用 Qt6 和 libghostty-vt 搭个最简终端。不要标签页，不要分屏，不要搜索，就一个 QWidget，能跑 bash，能执行命令，颜色和光标正常显示。

### 3.1 项目结构

```
minimal-terminal/
├── CMakeLists.txt
├── main.cpp
├── PtySession.h/.cpp
└── TerminalWidget.h/.cpp
```

`PtySession` 管 PTY 生命周期，`TerminalWidget` 管渲染和输入。边界清楚，后面扩展也方便。

### 3.2 PTY 与 Qt 事件循环

`forkpty()` 创建 PTY 后，父进程拿到 master fd。这个 fd 是阻塞的，别在主线程里轮询，会卡死 GUI。

Qt 的解法是 `QSocketNotifier`，把 Unix fd 的可读可写事件桥接到信号槽：

```cpp
m_readNotifier = new QSocketNotifier(m_masterFd, QSocketNotifier::Read, this);
connect(m_readNotifier, &QSocketNotifier::activated, 
        this, &PtySession::handleMasterReadyRead);
```

`handleMasterReadyRead()` 里 `read()` 读数据，通过 `dataReceived(QByteArray)` 抛给 `TerminalWidget`。

写入逻辑类似，但要处理大数据量的缓冲。`write()` 返回 `EAGAIN` 时注册 `QSocketNotifier::Write`，等 fd 可写了再刷缓冲区。不然一口气写几 MB，内核接收缓冲区满了，数据就丢了。

子进程这边 `forkpty()` 已经帮我们把 slave 绑到了标准 IO，直接 `execvp()` 启动 shell。shell 路径的查找顺序：先看 `$SHELL` 环境变量，再看 passwd 数据库，最后回退 `/bin/sh`。这个顺序是 POSIX 的习惯，照着做就行。

### 3.3 初始化 libghostty-vt

`TerminalWidget::initialize()` 里按顺序创建 handle：

```cpp
// 1. 终端状态机
GhosttyTerminalOptions opts = {
    .cols = m_cols,
    .rows = m_rows,
    .max_scrollback = 1000,
};
ghostty_terminal_new(nullptr, &m_terminal, opts);
ghostty_terminal_resize(m_terminal, m_cols, m_rows, cellWidth, cellHeight);

// 2. 注册 effect 回调
ghostty_terminal_set(m_terminal, GHOSTTY_TERMINAL_OPT_USERDATA, this);
ghostty_terminal_set(m_terminal, GHOSTTY_TERMINAL_OPT_WRITE_PTY, effectWritePty);
ghostty_terminal_set(m_terminal, GHOSTTY_TERMINAL_OPT_SIZE, effectSize);
ghostty_terminal_set(m_terminal, GHOSTTY_TERMINAL_OPT_TITLE_CHANGED, effectTitleChanged);

// 3. 渲染状态
ghostty_render_state_new(nullptr, &m_renderState);
ghostty_render_state_row_iterator_new(nullptr, &m_rowIter);
ghostty_render_state_row_cells_new(nullptr, &m_rowCells);

// 4. 输入编码器
ghostty_key_encoder_new(nullptr, &m_keyEncoder);
ghostty_key_event_new(nullptr, &m_keyEvent);
ghostty_mouse_encoder_new(nullptr, &m_mouseEncoder);
ghostty_mouse_event_new(nullptr, &m_mouseEvent);
```

几个我踩过的坑：

- `ghostty_terminal_resize()` 创建后马上调用，把像素尺寸同步进去。libghostty-vt 内部需要知道每个单元格占多少像素。
- Effect 回调必须是 C-linkage 的静态函数。库内部用裸函数指针调用，没有 this 指针。`GHOSTTY_TERMINAL_OPT_USERDATA` 就是用来传 `this` 的，回调里 `static_cast<TerminalWidget *>(userdata)` 转回来。
- 最常见的 effect 是 `writePty`。终端状态机收到某些查询序列（比如"你是什么终端""鼠标事件报告"）时，需要主动写回响应到 PTY。
- 析构时释放顺序必须反过来：`mouseEvent` → `mouseEncoder` → `keyEvent` → `keyEncoder` → `rowCells` → `rowIter` → `renderState` → `terminal`。顺序错了可能崩溃或者内存泄漏。

### 3.4 处理 PTY 数据

`PtySession` 收到数据后，`TerminalWidget::onPtyDataReceived()` 把它喂给状态机：

```cpp
void TerminalWidget::onPtyDataReceived(const QByteArray &data) {
    m_pendingPtyData.append(data);
    scheduleTerminalRepaint();
}
```

这里不立刻渲染，而是先攒到 `m_pendingPtyData`，用一个单发定时器做 coalesce。终端输出经常是突发的，`cat` 一个大文件几毫秒能涌进来几十 KB。每读一次都重绘，帧率直接爆炸。延迟 1~8 毫秒合并处理，能把多次更新 batch 成一帧，CPU 占用降很多。

定时器触发后：

```cpp
bool TerminalWidget::flushPendingPtyData(QRect *repaintRegion) {
    if (m_pendingPtyData.isEmpty()) return false;
    
    ghostty_terminal_parser_feed(m_terminal, 
        reinterpret_cast<const uint8_t *>(m_pendingPtyData.constData()),
        static_cast<size_t>(m_pendingPtyData.size()));
    
    m_pendingPtyData.clear();
    return true;
}
```

`ghostty_terminal_parser_feed()` 是核心入口。VT 序列解析、状态更新、光标移动、屏幕滚动，全在这个调用里搞定。

### 3.5 渲染：从状态到像素

`paintEvent()` 调 `renderTerminal()`：

```cpp
void TerminalWidget::renderTerminal(QPainter &painter) {
    // 1. 同步终端状态到渲染状态
    ghostty_render_state_update(m_renderState, m_terminal);
    
    // 2. 获取默认颜色
    GhosttyRenderStateColors colors;
    ghostty_render_state_colors_get(m_renderState, &colors);
    
    // 3. 判断是否需要全量重绘
    GhosttyRenderStateDirty dirtyState;
    ghostty_render_state_get(m_renderState, GHOSTTY_RENDER_STATE_DATA_DIRTY, &dirtyState);
    bool fullRedraw = (dirtyState == GHOSTTY_RENDER_STATE_DIRTY_ALL) 
                      || m_backBuffer.isNull();
    
    // 4. 离屏绘制到 back buffer
    QPainter backPainter(&m_backBuffer);
    
    // 5. 逐行遍历，只画脏行
    int y = 0;
    while (ghostty_render_state_row_iterator_next(m_rowIter)) {
        bool rowDirty = true;
        ghostty_render_state_row_get(m_rowIter, 
            GHOSTTY_RENDER_STATE_ROW_DATA_DIRTY, &rowDirty);
        
        if (!fullRedraw && !rowDirty) {
            y += m_cellHeight;
            continue;
        }
        
        renderRow(backPainter, y, colors);
        
        bool clean = false;
        ghostty_render_state_row_set(m_rowIter, 
            GHOSTTY_RENDER_STATE_ROW_OPTION_DIRTY, &clean);
        y += m_cellHeight;
    }
    
    // 6. 贴到屏幕
    painter.drawImage(contentOrigin, m_backBuffer);
}
```

增量渲染的关键在第 5 步：只重绘 dirty 的行。libghostty-vt 内部维护了每行的修改标记，`ghostty_render_state_update()` 会把这个信息同步过来。

`renderRow()` 的逻辑更细：

```cpp
void TerminalWidget::renderRow(QPainter &painter, int y, 
                               const GhosttyRenderStateColors &colors) {
    ghostty_render_state_row_get(m_rowIter, 
        GHOSTTY_RENDER_STATE_ROW_DATA_CELLS, &m_rowCells);
    
    // 整行背景填充
    painter.fillRect(0, y, width(), m_cellHeight, defaultBackground);
    
    int x = 0;
    while (ghostty_render_state_row_cells_next(m_rowCells)) {
        GhosttyCell rawCell;
        GhosttyStyle style;
        uint32_t graphemeLen;
        
        GhosttyRenderStateRowCellsData keys[] = {
            GHOSTTY_RENDER_STATE_ROW_CELLS_DATA_RAW,
            GHOSTTY_RENDER_STATE_ROW_CELLS_DATA_STYLE,
            GHOSTTY_RENDER_STATE_ROW_CELLS_DATA_GRAPHEMES_LEN,
        };
        void *values[] = {&rawCell, &style, &graphemeLen};
        ghostty_render_state_row_cells_get_multi(m_rowCells, 3, keys, values, &written);
        
        // 宽字符处理
        GhosttyCellWide cellWide;
        ghostty_cell_get(rawCell, GHOSTTY_CELL_DATA_WIDE, &cellWide);
        int cellWidth = (cellWide == GHOSTTY_CELL_WIDE_WIDE) 
                        ? m_cellWidth * 2 : m_cellWidth;
        
        // 颜色处理（反色、自定义前景背景）
        // ...
        
        // 绘制文字
        if (graphemeLen > 0) {
            QString text = extractText(m_rowCells, graphemeLen);
            painter.drawText(x, y + m_fontAscent, text);
        }
        
        x += cellWidth;
    }
}
```

实际代码里还有个 TextRun 合并的优化：相邻单元格字体和颜色相同的话，把文字拼成一段再 `drawText()`，减少绘制调用。满屏文本的时候这个优化很管用，帧率能差出几倍。

### 3.6 键盘输入编码

`keyPressEvent()` 里不能直接发 `QKeyEvent::key()` 给 PTY，终端程序要的是 VT 转义序列。按上方向键不是发字符 `'↑'`，而是发 `\x1b[A`。

```cpp
void TerminalWidget::keyPressEvent(QKeyEvent *event) {
    GhosttyKey ghosttyKey = mapQtKeyToGhostty(event->key());
    
    ghostty_key_event_set(m_keyEvent, GHOSTTY_KEY_EVENT_OPT_KEY, &ghosttyKey);
    ghostty_key_event_set(m_keyEvent, GHOSTTY_KEY_EVENT_OPT_MODS, &mods);
    ghostty_key_event_set(m_keyEvent, GHOSTTY_KEY_EVENT_OPT_UTF8, utf8Data);
    
    uint8_t buf[128];
    size_t len = 0;
    ghostty_key_encoder_encode(m_keyEncoder, m_keyEvent, buf, sizeof(buf), &len);
    
    if (len > 0 && m_ptySession) {
        m_ptySession->write(QByteArray(
            reinterpret_cast<const char *>(buf), static_cast<int>(len)));
    }
}
```

libghostty-vt 实现了 Kitty Keyboard Protocol，能区分 `Ctrl+C` 和 `Ctrl+Shift+C`，支持独立按键事件报告。跟老旧的 xterm 编码比，现代多了。

### 3.7 窗口大小变化

用户调整窗口时，三样东西要同步：

```cpp
void TerminalWidget::resizeEvent(QResizeEvent *) {
    // 1. 算新的行列数
    updateGridSize();  // m_cols = width / m_cellWidth
    
    // 2. 通知 libghostty-vt
    ghostty_terminal_resize(m_terminal, m_cols, m_rows, 
                            m_cellWidth, m_cellHeight);
    
    // 3. 通知 PTY（让 stty size 和 SIGWINCH 生效）
    m_ptySession->resize(m_cols, m_rows, m_cellWidth, m_cellHeight);
}
```

`PtySession::resize()` 里用 `ioctl(fd, TIOCSWINSZ, &winsize)` 把新尺寸设到 PTY。shell 和它的子进程收到 `SIGWINCH` 信号，vim、htop 这些 TUI 程序就知道该重绘了。

实际实现里 resize 会做防抖，`QTimer` 延迟几十毫秒再应用。拖拽窗口时如果每像素都刷新，闪烁会很严重。

---

## 四、从"能跑"到"好用"，还有多少距离

上面这个最小终端能跑 shell、执行命令、显示颜色、处理键盘。但变成日常可用的生产级终端，要做的事情一大堆。

### 4.1 文本交互

- 鼠标选择
  - 单点、双击选词、三击选行
  - 选区与屏幕坐标的映射（考虑滚动偏移、宽字符占两列的情况）
  - 矩形选择模式
- 复制与粘贴
  - 系统剪贴板集成（`QClipboard`，PRIMARY 中键粘贴 / CLIPBOARD Ctrl+C/V 两种 Linux 惯例）
  - bracketed paste mode，防止粘贴内容被 shell 当成命令直接执行
  - 选区文字提取，跨行时要处理 wrap 状态

### 4.2 滚动与滚动缓冲区

- 鼠标滚轮事件转成滚动行数
- 滚动条位置与 viewport 的同步
- 滚动缓冲区行数限制，避免内存无限增长
- 快速滚动优化，批量跳过而不是逐行处理

### 4.3 搜索

- 在滚动缓冲区里正向/反向搜文本
- 匹配结果的高亮渲染
- 当前匹配项的跳转标识

### 4.4 Shell 集成

- OSC 序列解析
  - 窗口/标签标题（`OSC 0/2`）
  - 当前执行命令和退出码（`OSC 777` 自定义、`OSC 633` VS Code 协议、`OSC 1337` WezTerm 协议、`OSC 133` FinalTerm 协议）
  - 当前工作目录追踪
- Shell 集成脚本注入
  - 为 bash/zsh/fish 生成预加载脚本
  - 临时目录的创建和清理
- 命令状态追踪（Idle / Running / Succeeded / Failed）
- 超链接检测（`OSC 8`）

### 4.5 主题、字体与视觉效果

- 主题系统（ANSI 色板、前景/背景色、选择高亮色）
- 透明背景与窗口合成器模糊
- 光标样式（块 / 线 / 下划线）和闪烁控制
- 字体回退与特殊符号（emoji、Powerline、Nerd Fonts）
- 粗体、斜体、下划线、删除线渲染

### 4.6 现代终端协议

- Kitty Keyboard Protocol
- SGR Mouse 格式
- Focus Event（焦点变化通知）
- Unicode 宽字符与 grapheme cluster
- 连字字体（ligatures）
- 图像协议（Sixel / Kitty Graphics / iTerm2 inline images）

### 4.7 应用层架构

- 多标签页管理（创建、关闭、切换、标题同步）
- 分屏（水平/垂直分割、焦点切换、快捷键）
- 远程管理（SSH 连接配置面板）
- 设置对话框与配置持久化
- 全局快捷键与局部快捷键的优先级处理

### 4.8 性能优化

- 增量渲染与脏行跟踪（libghostty-vt 给了基础能力，UI 层要用对）
- PTY 数据合并（coalesce 机制）
- 光标局部重绘
- 双缓冲（`QImage` 离屏绘制）
- Resize 防抖
- 字体缓存（预计算 `QFontMetrics`，缓存粗体/斜体变体）
- 大缓冲区下的内存与渲染策略

### 4.9 输入法与国际化

- 输入法预编辑文本（preedit）的显示
- `QInputMethod` 光标位置同步
- 复杂文本布局（RTL、组合字符）

### 4.10 测试与质量保障

- 单元测试
  - PTY 会话生命周期
  - 终端尺寸计算
  - 键盘映射与编码
  - 渲染状态脏跟踪
- 自动化测试
  - esctest 验证 VT 序列兼容性
  - 终端行为回归测试
- 兼容性测试
  - 不同 shell 的 Shell 集成验证（bash/zsh/fish）
  - TUI 程序兼容性（vim、tmux、htop、ncdu 等）
  - 鼠标协议兼容性
  - 各种终端应用的键盘交互
- 性能测试
  - 大文件 cat 的帧率
  - 滚动缓冲区内存占用
  - 长时间运行稳定性
- 崩溃处理
  - 信号捕获与崩溃日志收集
  - 核心转储分析

这个清单里，deepin-terminal-ghostty 目前只做了大约一半。终端模拟器这个品类，表面看就是"一个黑框里跑命令"，真要做到顺手，细节多得吓人。每个"小细节"背后可能都藏着几个星期的调试，想想还有点小兴（pi）奋（bei）呢。
