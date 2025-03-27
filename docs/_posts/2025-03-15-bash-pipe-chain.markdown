---
layout: default
title:  "管道与 SIGPIPE 中的坑"
date:   2025-03-15 00:00:00 +0800
categories:
---

熟悉 bash 编程的人都知道, 管道的读端关闭后, 写端会收到 `SIGPIPE` 信号, 并且默认会导致进程退出.
这使我们能做到如下的事情:
```bash
foo | head
```
我们希望的结果是
`head` 读取了 `foo` 输出的前几行后退出 (并关闭 `stdin`),
导致 `foo` 收到 `EPIPE` 错误退出.

但实际真的这么美好吗, 来看看下面这个例子.
假设我们需要让一个命令这里以 `cat` 为例在输出第一行后退出, 一个很自然的想法是:
```bash
cat | head -n1
hello
hello
world
```
但实际测试会发现, `cat` 并未在输入第一个 `hello\n` 后退出, 而是继续读入了第二行 `world\n`.

原因也十分简单, 读进程在尝试写入一个另一端关闭的管道或 socket 才会收到 `SIGPIPE`, 
```
$ strace -ewrite cat | head -n1
hello
write(1, "hello\n", 6)                  = 6
hello
world
write(1, "world\n", 6)                  = -1 EPIPE (Broken pipe)
--- SIGPIPE {si_signo=SIGPIPE, si_code=SI_USER, si_pid=161457, si_uid=1000} ---
+++ killed by SIGPIPE +++
```
可以看到, `cat` 在写入 `world\n` 时才收到了 `SIGPIPE`,
要是没有第二行写入, `cat` 就无从得知 `head` 已经退出了.

---

你可能会说这虽然不符合预期, 但如果我们不需要第一行的内容就还是有办法解决的,
比如让管道的读端直接关闭:
```bash
cat | head -n0
```
或者
```bash
cat | true
```
这样只要 `cat` 一写入就会 `SIGPIPE`, 同样达到了目的.

然而当你遇到一个管道链时, 这种 hack 就不奏效了:
```bash
cat | cat | cat | cat | true # (实际这里的 `cat` 可能是 `grep`, `awk`, `sed` 等等)
```
输入的每一行都只能导致一个 `cat` 退出, 而无法终止整个管道链.

---

另一个能用的方法是直接使用 subshell
```bash
(foo | bar | { head -n1; exit 0; } ) # 也许是 `kill -INT $$`
```
