# v2.2.2-coloros.2 发布说明

**一个会让手机持续发烫的严重 bug 修复，强烈建议 ColorOS 版用户全部升级。**

## 问题

`cirno_daemon` 的主循环几乎不睡觉，长期占满一个 CPU 核心，导致手机明显发热。

## 根因

`src/cirno_daemon.cpp` 里主循环的小睡写成了：

```cpp
struct timespec ts = {0, 1000 * 1000 * 1000};   // tv_nsec = 1,000,000,000
nanosleep(&ts, nullptr);
```

POSIX 要求 `tv_nsec` **严格小于 10⁹**（即必须小于 1 秒）。传 10⁹ 是非法参数，`nanosleep()` 不做任何等待、立刻返回 `EINVAL`，而返回值没有被检查，循环就退化成纯空转。

在一台 ColorOS 17 真机上实测：

| 指标 | 修复前 | 修复后 |
|---|---|---|
| `cirno_daemon` CPU 占用 | **99%~103%（单核满载）** | **0%** |
| 累计 CPU 时间 | 227 分钟里占了 172 分钟 | 几乎为 0 |
| 系统调用速率 | **2707 次/秒** | 0（空闲时） |
| 非自愿上下文切换 | 164 万次 | 十余次 |
| CPU 结温（大核传感器） | **81~96 °C** | **55 °C** |
| 主板 sensor（board2-ntc） | 61.8 °C | 50.3 °C |
| 电源芯片（ap_ntc） | 57.0 °C | 47.8 °C |

验证方式（本机复现，`{0, 1e9}` 对 `nanosleep` 的实际行为）：

```
OLD {0,1e9}: ret=-1 errno=22 elapsed=0.000001s  -> NO SLEEP (busy loop)
NEW sleep_seconds(1.0): elapsed=1.001s
NEW sleep_seconds(0.2): elapsed=0.201s
```

## 修复内容

1. **新增 `sleep_seconds(double)` 封装**：把「秒数」规范化为合法的 `tv_sec` / `tv_nsec`（自动进位，保证 `tv_nsec < 1e9`），正确处理 `EINTR` 重试（用 `nanosleep` 的剩余时间参数续睡），并在 `g_stop` 置位时提前返回。主循环不再直接拼 `timespec`。
2. **主循环改为睡到下一个待办事件**：不再无脑固定 1 秒，而是计算心跳、定期回收、大进程看门狗、前台舒缓中最近的一个时间点，最长睡 2 秒、最短 0.2 秒。既修了 bug，也减少了无意义的唤醒（原来即使什么都不做，也要每秒醒一次）。
3. **另外两处 `nanosleep` 一并替换**：`--once` 模式的 0.2 秒轮询、退出清理的 0.1 秒等待。这两处的数值本身合法，但统一走封装可以根除同类问题。

修复后功能行为完全不变：`reclaim` 仍按 `interval` 正常调度，`SKIP reason=mem_ok` 判定正常，心跳持续更新，watchdog 照常工作，模块无需停用或降级。

## 顺带说明

- `bin/cirno_daemon` 由 `src/cirno_daemon.cpp` 用 zig 0.13.0 重新编译（`-target aarch64-linux-musl -static -Os -s`），与仓库既有产物的构建方式一致。
- `bin/cirno_pageout` 未改动（源码与产物均未变，可复现构建校验通过）。
- 只有 `src/cirno_daemon.cpp`、`bin/cirno_daemon` 两个文件的实质内容变化，其余文件仅版本号变化。

## 升级方式

在 Magisk / KernelSU / APatch 里直接刷入新的模块 zip 覆盖安装即可，配置（`/data/adb/cirno_reclaim/config.conf`）会被保留。

升级后建议这样确认已经生效：

```sh
# 守护进程的 CPU 占用应该在 0% 左右，而不是接近 100%
top -b -n 1 -m 5 | grep cirno_daemon
```

如果你之前因为发烫而临时停用了模块，升级后可以重新启用。

## 校验

```sh
sha256sum -c CHECKSUMS.txt
```