# PLAN: MiTV-MTEQ0 开机接管竞态

状态：**待做**。诊断已完成，改动未开始。起新 session 直接从「改法」一节动手。

## 问题

小米电视 ES75 2022 款（`MiTV-MTEQ0`，代号 `baymax`，Android 10，固件 `22.6.27.2363`）装上 Aerial 1.22 后，**开机不会自动进 Aerial**，停在 PatchWall。按一次 Home 才过去。待机唤醒、手动进 Aerial、守护绑定、`KEEP_ADB` 都正常。

## 时序证据（2026-10-22 一次干净重启）

日志 tag `AerialGuard`，配合 `am_set_resumed_activity`：

```
01:12:48.956  AerialGuard: revived by the notification listener binding
01:12:49.712  am_set_resumed_activity: com.mitv.tvhome/.MainActivityUserMode
01:12:50.652  AerialGuard: back after Aerial died in front with the stock launcher in front; bringing Aerial back
01:12:51.206  am_set_resumed_activity: app.orchard.tv/.MainActivity          ← Aerial 上前台
01:12:52.017  am_set_resumed_activity: com.mitv.tvhome/.BootActivity          ← PatchWall 插队
01:12:52.110  MitvUserUsageStatsService: last=app.orchard.tv,now=com.mitv.tvhome
01:12:53.515  am_set_resumed_activity: com.mitv.tvhome/.MainActivityUserMode  ← 回到原桌面
01:12:56.211  AerialGuard: user unlocked with app.orchard.tv in front; leaving it there  ← 误判
01:13:03.666  AerialGuard: ADB debugging had been switched off; turned it back on
```

结果：焦点停在 `com.mitv.tvhome/.MainActivityUserMode`，Aerial 在后台。

## 根因

两个因素叠加：

1. **`stockHomes` 名单不全。** `refreshStockHomes()`（`HomeGuardService.kt`）只收 `CATEGORY_HOME` 的 activity，也就是 `com.mitv.tvhome/.MainActivityUserMode`。PatchWall 的 `com.mitv.tvhome/.BootActivity` 不在名单里。`onAccessibilityEvent` 里这行把它放行了：

   ```kotlin
   if (cls !in stockHomes.getValue(pkg) || allowed) return
   ```

   于是 `BootActivity` 一出现，守护不接管，`lastWindow` 被更新成 `(com.mitv.tvhome, BootActivity)` 之外的东西（或者干脆没更新，见下一条）。

2. **`lastWindow` 在竞态窗口里不可信。** 01:12:56 那句 `user unlocked with app.orchard.tv in front` 说明 `lastWindow` 还停在 Aerial，可实际前台已经是 `MainActivityUserMode`。01:12:53 那次 `resumeTopActivityInnerLocked` 的 `TYPE_WINDOW_STATE_CHANGED` 没有送达守护（或 `isActivity()` 判否）。所以 `takeOverIfStockHomeInFront()` 里 `stockInFront = last == null || last.first in stockHomes` 算出 false，就地返回。

`BOOT_WINDOW_MS` 是 3 分钟，这次完全在窗口内，不是窗口过期的问题。

另有一个会干扰排查的干扰项：`GuardSetup.revivedOften()`（10 分钟内复活 3 次就判定「固件反复杀 Aerial」并主动让位，防乐视 eUI，issue #63）。**安装、`repair`、`force-stop` 这些操作都会写复活计数**，调试期很容易凑够 3 次，导致守护在开机时直接罢工，症状和竞态一样。排查前先等 10 分钟让计数过期，或者看日志里有没有 `keeps being stopped` 那句。

## 改法

推荐**改动最小、且不动 HDMI-CEC 开机进信号源那条路径**的方案：

**接管后再补一次延迟复查。** `takeOverOnceSettled()` 已经有 `SETTLE_MS = 1500` 的延迟机制，在 `takeOverIfStockHomeInFront()` 真的 `goHome()` 之后，再排一次 `takeOverOnceSettled(reason + " (recheck)")`，间隔放大到 3 到 4 秒。这样 PatchWall `BootActivity` 抢跑也会被第二次检查看到。

好处：
- 不改 `stockHomes` 的语义，不影响「用户主动打开原桌面」的 `allowStockUntil` 逻辑
- 不影响 `hasBootSource()`（开机自动进信号源）那条分支
- 对其他机型是幂等的：如果第二次检查时 Aerial 已在前台且 `hasBootSource()` 为真，走的是 `goHome(woke = true)` 补发唤醒标记，正是 issue #111 在 MTEQ0 上要的行为

备选方案（可以顺手一起做）：在 `refreshStockHomes()` 里把同一包名下、非 `CATEGORY_HOME` 的 activity 也算进 `stockHomes` 的 class 集合。注意 `com.mitv.tvhome/.activity.StrategicLayerActivity` 的注释里已经提到过类似情况，但当前实现仍然只认 HOME 那个类。这条要小心：用户主动从 Aerial 的「桌面」磁贴进 PatchWall 时走的是 `GLOBAL_ACTION_HOME`，`allowStockUntil` 窗口内不应被拉回，改 class 集合时别破坏这个分支。

不要做的：停用 `com.mitv.tvhome`（`AGENTS.md` 硬规则，会卡 FallbackHome 黑屏）。

## 验证

改完必须真机验，模拟器上没有 PatchWall 的 `BootActivity`：

1. `./gradlew assembleStandardRelease`，`BUILD=1 KEEP_ADB=1 TV=192.168.1.9 tools/install.sh`
2. `adb shell am force-stop app.orchard.tv`，等 15 秒（躲开 `justUpdated()` 的 2 分钟窗口，否则 `revived` 分支会被跳过）
3. `adb reboot`，等开机，跑验收：
   ```
   adb shell "dumpsys window | grep mCurrentFocus"
   ```
   期望焦点是 `app.orchard.tv/app.orchard.tv.MainActivity`，不需要按 Home
4. 反向验一次 `allowStockUntil`：从 Aerial 的应用网格里打开 PatchWall，确认不会被立刻拉回
5. 如果配了「开机时打开」的信号源，验 HDMI-CEC 那条没被破坏
6. 待机 1 分钟以上再唤醒，确认仍是 Aerial（小米会强停第三方应用）

调试技巧见 `docs/HANDOFF.md` 的「调试技巧」一节，守护日志 tag 是 `AerialGuard`。

## 上游同步

值得给 `pengchujin/aerial-tv` 报 issue，附本文的时序日志。`docs/devices/` 下已有 `xiaomi-mitv-mftr0.md`、`xiaomi-mitv-mftp0.md`、`xiaomi-mitv-mffu0.md`，**缺 `xiaomi-mitv-mteq0.md`**，正好按那个格式补一份（ES75 2022 / EA Pro 同平台，`ro.product.model` 都是 `MiTV-MTEQ0`）。

`docs/HANDOFF.md` 的「现状」一节提到 MTEQ0 只验过 issue #111 的待机唤醒改法，没验开机接管，这条缺口可以一并说明。

## 本机事实

| 项 | 值 |
|---|---|
| 电视 | 小米电视 ES75 2022 款 |
| `ro.product.model` | `MiTV-MTEQ0` |
| `ro.product.name` | `baymax` |
| `ro.board.platform` | `mt5872` |
| `ro.build.fingerprint` | `xiaomi/baymax/baymax:10/QQ2A.200305.004.A1/2363:user/release-keys` |
| 固件 | `22.6.27.2363`，Android 10（SDK 29） |
| 分区 | 单分区（`ro.boot.slot_suffix` 空） |
| 电视 IP | `192.168.1.9`（ADB 端口 5555） |
| 已装 Aerial | 1.22（versionCode 24），standard 版，`KEEP_ADB=1` |
| adb 路径 | `~/tools/platform-tools-latest/platform-tools/adb` |

注意：小米会主动关 ADB，Aerial 的 `KEEP_ADB=1` 会自动打开。连不上时先看 `adb connect 192.168.1.9:5555`。

`local/` 目录不存在，没有签名密钥。编译会回落到 debug key，但因为这台电视上是第一次装 Aerial（不存在签名不一致覆盖安装的问题），debug key 也能装。要发版给别人用才需要 `local/signing.*`。
