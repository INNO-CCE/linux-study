# 第 1 周任务清单

> **周期**：2026.10.10（周六）— 10.18（周日），共 9 天
> **本周主题**：把工程工具链变成肌肉记忆（Git / Makefile / GDB），并暴露 C 语言的漏洞
> **本周唯一目标**：GitHub 上有一个能编译、能调试、有 6 次以上提交的多文件 C 项目

---

## 一、起点盘点

| 项目 | 状态 |
| :-- | :-- |
| Ubuntu 22.04 原生系统 | ✅ 已完成，原规划第 1 周的装系统任务**跳过** |
| gcc 12.3 / gdb / make / git | ✅ 已装 |
| VSCode | ✅ 已装（`code .` 可用） |
| git 身份（user.name / user.email） | ❌ 未配置 |
| SSH key | ❌ 无（`~/.ssh` 为空） |
| GitHub 仓库 | ❌ 无 |
| 开发板 / 串口工具 / 交叉工具链 | ❌ 无 —— **本周不买**，见 `PLAN-REVIEW.md` 第二节第 3 条 |

**本周不要做的事**：不要买开发板、不要碰内核驱动、不要看韦东山视频、不要配 Yocto。这周只做工具链和 C，做扎实比做多重要。

---

## 二、逐日任务

### D1 · 10.10（周六，约 2h）— 工程基础设施

这是唯一一天以「配置」为主的日子，一次配好，三年受益。

**1. Git 身份与 SSH key**

```bash
git config --global user.name  "你的名字"
git config --global user.email "你的邮箱@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim        # 或 code --wait
git config --global pull.rebase true
git config --global alias.lg "log --oneline --graph --decorate --all"

ssh-keygen -t ed25519 -C "你的邮箱@example.com"   # 一路回车
cat ~/.ssh/id_ed25519.pub                          # 复制输出
```

把公钥贴到 GitHub → Settings → SSH and GPG keys → New SSH key。

**2. 建仓库并推上去**

```bash
cd /home/fracc/dsh/linux-study
git init
printf '*.o\n*.d\n*.out\na.out\ncore\n.vscode/\n' > .gitignore
git add .
git commit -m "docs: 初始规划与评审"
# 在 GitHub 网页建一个空仓库 linux-study（不要勾 README/gitignore）
git remote add origin git@github.com:你的用户名/linux-study.git
git push -u origin main
```

**✅ 验收**
- `ssh -T git@github.com` 返回 `Hi xxx! You've successfully authenticated`
- GitHub 网页上能看到 `README.md` 和 `docs/`
- `git lg` 有输出

---

### D2 · 10.11（周日，约 4h）— C 语言压力测试

**不要「复习」，要「做题暴露问题」。** 下面 5 题**先不许查资料**，卡住 20 分钟以上再查，查完把「为什么卡住」写进笔记。

1. **通用交换**：写 `void swap(void *a, void *b, size_t size)`，要能交换任意类型。想想为什么参数里必须有 `size`。
2. **函数指针做策略**：写一个排序函数，排序算法（冒泡/插入）通过函数指针参数传入。再用一个函数指针**数组**实现一个简单的状态机。
3. **内存布局**：定义 `struct S { char c; int i; double d; };`，用 `printf` 打印 `sizeof(struct S)` 和每个成员的 `offsetof`，**然后解释**：为什么不是 13？如果把 `double d` 挪到最前面，大小会变吗？
4. **手写单链表**（不查资料，20 分钟内完成）：创建、尾部插入、删除指定值、反转、找中间节点。要求处理「删除头节点」和「空链表」。
5. **故意制造 3 个 bug**：空指针解引用、数组越界 1 个字节、`free` 两次。先用**肉眼**猜哪一行会崩，再用 GDB 验证（D5 会用）。

**✅ 验收**
- 5 道题都有代码，提交到 `c-basics/` 目录
- 第 3 题能口头解释对齐规则（面试原题）
- 第 4 题反转链表的代码，你能在纸上画指针变化过程

---

### D3 · 10.12（周一，1.5h）— Makefile 三步走

**关键：不要抄一个 Makefile，要分三步亲手演化，你会看到每一步解决了什么问题。**

```bash
mkdir -p week1/makefile-lab && cd week1/makefile-lab
# 建 3 个文件：main.c、foo.c、foo.h
```

- **第 1 步**：写最笨的显式规则版（每个 `.o` 一条规则，手写依赖）。感受痛点：加一个文件要改三处、头文件改了不重编。
- **第 2 步**：改造成变量（`CC`、`CFLAGS`、`OBJS`、`TARGET`）+ 自动变量（`$@ $< $^`）。感受痛点：十条规则长得一样。
- **第 3 步**：改造成模式规则 `%.o: %.c` + 自动依赖 `CFLAGS += -MMD -MP` + `-include $(DEPS)` + `.PHONY: all clean`。

**必须理解的 6 个点**（面试会问）：
`$@` `$<` `$^` 分别是什么 · 为什么 `.PHONY` 重要 · `-MMD -MP` 生成什么文件、解决什么问题 · `=` 和 `:=` 的区别 · `-Wall -Wextra -Werror` 各做什么 · 为什么 `clean` 要用 `rm -f`

**✅ 验收**
- `make clean && make` 零警告通过
- 改一次 `foo.h` 后 `make`，**只重编受影响的文件**（观察输出确认）
- 能不看资料从零写出第 3 步版本

---

### D4 · 10.13（周二，1.5h）— GDB 实战

用 D3 的代码，编译时加 `-g -O0`。练到下面每条都不查手册：

```
break / b          tbreak（临时断点）
run / r            continue / c
next / n           step / s
print / p          p *ptr   p arr[3]   p/x var
info locals        info args        info breakpoints
backtrace / bt     frame N          up / down
watch var          display var      x/16xb ptr
```

**必做实验：**
1. 在函数入口打断点，`bt` 看调用栈，`frame` 切换并 `p` 出每层的局部变量。
2. `watch` 一个变量，看它在哪里被意外修改。
3. **core dump 三件套**：
   ```bash
   ulimit -c unlimited
   # 让程序崩一次，生成 core 文件
   gdb ./a.out core
   bt          # 直接看到崩溃点
   ```

**✅ 验收**
- 给一个段错误的程序，你能在 **60 秒内**用 GDB 说出崩溃的文件和行号
- 理解 `-g` 和 `-O0` 为什么调试时要加（试试 `-O2` 调试，感受变量被优化掉的困惑）

---

### D5 · 10.14（周三，1.5h）— 用调试器「看」内存

用 D2 的链表代码，在 GDB 里：

```
p head              p *head             p head->next
x/32xb head         观察节点在堆上的真实字节布局
p &head->data       p sizeof(struct node)
```

**做出的结论（写进笔记）**：
- 一个 `struct node` 在 64 位系统上占多少字节？为什么不是各成员之和？
- 链表节点的地址是连续的还是跳跃的？这解释了链表相对于数组的什么特性？
- `free` 之后那块内存的**前 16 个字节**变成了什么？（这是 glibc 的 `tcache`/`malloc_chunk` 元数据，看一眼印象深刻）

**✅ 验收**：能在 GDB 里画出你的链表在内存中的实际拓扑图（手绘即可，拍照存进仓库）

---

### D6 · 10.15（周四，1h）— 项目立项

**本周项目：`minitool` —— 一个会连续进化 4 周的命令行工具。**

不要做「Hello World 集合」，要做一个**能长大**的项目。设计如下：

| 周次 | 版本 | 新增能力 | 面试可讲 |
| :-- | :-- | :-- | :-- |
| **W1** | v0.1 | 多文件 C 项目 + Makefile + GDB 调试 | 工程规范 |
| W2 | v0.2 | 用 POSIX 系统调用（`open/read/write`）重写，完整 `errno` 处理 | 系统调用、错误处理 |
| W3 | v0.3 | 多进程/多线程并行处理多个文件 | `fork`/`pthread`、并发 |
| W4 | v0.4 | 加 Socket，变成 C/S 架构的远程工具 | 网络编程 |

**功能选什么**（挑一个，建议选 1）：
1. **文本统计工具**（`wc` 简化版）：统计行数/单词数/字符数，支持多文件、支持 `-l -w -c` 选项。**推荐**，因为 W2—W4 能自然叠加。
2. 一个迷你动态数组 + 链表的库，带 CLI 演示程序。
3. 十六进制查看器（`xxd` 简化版）。

**项目结构（一开始就按这个来）：**
```
minitool/
├── Makefile
├── README.md          # 项目说明 + 编译方法 + 使用示例
├── include/
│   └── minitool.h
├── src/
│   ├── main.c         # 参数解析
│   ├── counter.c      # 核心统计逻辑
│   └── io.c           # 文件读写
├── tests/
│   └── run_tests.sh   # 哪怕只有 5 行断言也要有
└── docs/
    └── debug-notes.md # 调试记录（这个比代码更值钱）
```

**今天只做**：建目录、写 `README.md`、写头文件、把 `main.c` 的空架子跑通并提交。**不要一天写完，要让 Git 历史有层次。**

**✅ 验收**：`git commit` 一次，内容是项目骨架

---

### D7 · 10.16（周五，1h）— 实现核心逻辑

实现 `-l`（行数）和 `-c`（字符数）功能，命令行参数用 `getopt()` 解析。

**注意**：这周先用**标准 C 库**（`fopen`/`fgets`/`fgetc`），**故意不用系统调用**。因为下周你要重写它，那时你才会真正体会 `fopen` 和 `open` 的区别 —— 这个对比是下周博客的素材。

**✅ 验收**：`./minitool -l README.md` 输出与 `wc -l README.md` 一致

---

### D8 · 10.17（周六，4h）— 收尾 + 打磨

1. 补齐 `-w`（单词数），写完 `tests/run_tests.sh`，让 `make test` 能跑。
2. 打开 `-Wall -Wextra -Werror`，**消灭所有警告**（这一步往往比写功能更花时间，但必须做）。
3. 用 GDB 走一遍完整流程，把过程记进 `docs/debug-notes.md`。
4. 补 `README.md`：项目简介、编译、用法示例、一个「已知问题」章节。**「已知问题」章节很重要** —— 它证明你在诚实思考，面试时也是很好的话题引子。

**✅ 验收**：`make clean && make && make test` 一条龙通过，零警告

---

### D9 · 10.18（周日，4h）— 第一篇文章 + 复盘

**1. 写本周博客（≥800 字，problem-driven）**

从下面挑一个你真实卡住过的问题，**不要写知识总结**：

- 「我改了一个头文件，为什么 `make` 没有重编？—— 从零搞懂 `-MMD -MP`」
- 「GDB 里 `p` 出来的变量显示 `<optimized out>`，我折腾了半小时才明白是 `-O2` 干的」
- 「`malloc` 一块 4 字节内存，`free` 之后前 16 字节变成了什么」
- 「我的 Makefile 明明写对了，为什么 `make clean` 报 `No rule to make target`」

文章结构：**现象 → 我的错误猜测 → 排查过程 → 真正原因 → 如何避免**。发到 GitHub `notes/blog/` 或任意平台。

**2. 打开 `PLAN-REVIEW.md` 第五节的「能力自检清单」**，逐条标记：✅ 能讲清 / ⚠️ 见过但讲不清 / ❌ 不会。**这个记录每季度做一次，就是你三年的成长曲线。**

**3. 写 3 行周复盘**（存进 `notes/weekly/2026-W41.md`）：
- 本周实际投入多少小时？
- 哪件事比预想的难？
- 下周要砍掉什么？

---

## 三、本周总验收标准

全部满足才算过关（这是可以客观检验的，不要自我欺骗）：

- [ ] `ssh -T git@github.com` 认证成功
- [ ] GitHub 上有 `linux-study` 仓库，且**本周提交 ≥6 次**，commit message 有意义（不是 `update`）
- [ ] `minitool` 项目 `make clean && make` 在 `-Wall -Wextra -Werror` 下**零警告**
- [ ] 能**不看资料**写出带模式规则 + 自动依赖的 Makefile
- [ ] 能用 GDB 在 **60 秒内**把一个段错误定位到具体行号
- [ ] 能在纸上画出单链表反转的指针变化过程
- [ ] `struct {char c; int i; double d;}` 的 `sizeof` 是 16，你能解释为什么（以及重新排序后是多少）
- [ ] 1 篇 ≥800 字博客，主题是**排查一个具体问题**
- [ ] `docs/debug-notes.md` 有内容
- [ ] 完成一次能力清单自评

**如果只完成了 60%** —— 不要延长这周。直接进入第 2 周，把没做完的挂到 `notes/backlog.md`。**学习最大的敌人不是进度慢，是停下来补作业。**

---

## 四、下一周预告（先不要做，只是让你有数）

第 2 周主题：**Linux 系统编程 I —— 文件 I/O 与 errno**

- 用 `open`/`read`/`write`/`lseek`/`close` 重写 `minitool` 的 IO 层（→ v0.2）
- 搞清 `FILE*` 和 `fd` 的关系、什么是缓冲区、`fopen` 内部做了什么
- `errno` + `perror` + `strerror` 的正确用法
- 理解 `O_CREAT` 的第三个参数（和权限掩码 `umask`）
- 学会用 `strace` 黑盒观察程序的所有系统调用
- 博客选题：从 `fopen` 换成 `open` 后程序变快了/变慢了，为什么？

**提前准备**：`sudo apt install manpages-dev strace ltrace valgrind`（第 2 周要用）
