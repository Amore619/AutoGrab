[README.md](https://github.com/user-attachments/files/34192017/README.md)# AutoGrab — Android 无障碍服务自动化点击（求大佬指点）

一个基于 `AccessibilityService` 的 Android 自动点击工具。设计目标是：**在用户手动停在目标 APP 的某个列表页后，自动寻找并点击指定文本的按钮，找不到就自动滚动刷新。**

> ⚠️ 本项目**目前没有完全跑通**，卡在多个 Android 无障碍与 H5 WebView 兼容性问题上。发布目的是求社区帮助排查。欢迎 PR / Issue。

## 功能设计（部分已实现）

| 功能 | 状态 |
|---|---|
| 状态机驱动的自动导航（入口A → 入口B → 入口C） | ⚠️ 文本可读时可用，图片按钮不行 |
| 页面识别（启动时判断当前在哪一页，从对应步骤接续） | ✅ 已实现 |
| 文本节点查找 + 点击（含「不可点父节点向上找」「坐标兜底」） | ✅ 已实现，但 H5 页面文字读不到时无效 |
| 滚动：`ACTION_SCROLL_FORWARD` + `dispatchGesture` 双路轮替 | ⚠️ 两条路单独都能返回成功，但页面是否真的滚了不稳定 |
| 找不到目标按钮时每 10 秒滚动刷新 | ⚠️ 依赖上面的滚动 |
| 点击后的确认弹窗自动处理 | ✅ 已实现 |
| 成功后震动 + `IMPORTANCE_HIGH` 横幅通知 + 本地记录 | ✅ 已实现 |
| 「离开模式」：调暗系统亮度 + 延时跳转目标 APP + 常驻状态通知 | ✅ 已实现 |
| 设备管理器自动关屏 | ✅ 已实现 |
| 故障容错（先识别当前页，真的不在目标页才按返回键） | ✅ 已实现 |

## 技术栈

- 纯 Java，无第三方依赖（手动 aapt2 + d8 + apksigner 构建，也兼容 Gradle）
- `minSdk 24` / `targetSdk 34`
- 目标 APP 是 **H5（WebView）套壳应用**，这是大部分问题的根源

## 未解决的问题（求指点）

### 1. WebView 内容对无障碍不可见（最关键）

目标 APP 的首页核心入口是一张**图片卡片，文字绘制在图片里**，无障碍树中只有 ImageView 节点，`findAccessibilityNodeInfosByText()` 完全搜不到。已尝试：

- ❌ 文本匹配（读不到）
- ✅ 按屏幕比例坐标 `dispatchGesture` 点击（能用，但依赖分辨率/布局，换个机型就偏）

**问题：有没有更稳的办法点击 WebView 里图片上的内容？**（`ACTION_CLICK` 在这个 WebView 上对大部分节点无效，疑似 H5 未开启无障碍支持）

### 2. `ACTION_SCROLL_FORWARD` 返回 true 但页面不动

```
node.isScrollable() == true（WebView 节点）
node.performAction(ACTION_SCROLL_FORWARD) == true
但页面内容不滚动
```

而同一位置用 `dispatchGesture` 模拟滑动，**时灵时不灵**：
- 有一次实测能滚（间隔 1.5 秒连续滑动）
- 把间隔改成 0.8 秒连续滚就完全不动了
- 手势回调有时 `onCompleted`，页面也没动

**问题：WebView 的滚动到底该怎么做？是手势参数问题（时长/轨迹插值）还是需要先 `ACTION_SCROLL_FORWARD` 激活？**

### 3. H5 页面文字时有时无

同一个 H5 页面：
- 有时能通过 `findAccessibilityNodeInfosByText("入口B")` 找到文字节点
- 有时整棵树只剩 WebView 壳，一个文字都读不到

怀疑与 WebView 的无障碍事件初始化时机有关。**问题：是否需要先给 WebView 发某个事件唤醒它的无支持？或者等 `TYPE_WINDOW_CONTENT_CHANGED` 事件后再读？**

### 4. 小米 HyperOS 上的额外问题

- 无障碍服务会被系统悄悄杀掉/关闭（已引导用户锁后台 + 省电无限制）
- 通知渠道 `IMPORTANCE_MIN` 一旦创建，代码改成 `HIGH` 也不生效（渠道设置被系统缓存），只能换 channelId 重建
- `dispatchGesture` 在部分页面疑似被系统限制

## 项目结构

```
app/src/main/java/com/example/autoclicker/
├── MainActivity.java      主界面：权限引导、自检、离开模式开关
├── GrabService.java       核心：状态机 + 节点查找 + 滚动 + 通知
├── RecordsActivity.java   任务记录列表
├── HelpActivity.java      内置使用说明
└── SleepAdmin.java        设备管理器（仅用于自动关屏）
```

## 构建

```bash
# 需要 Android SDK (platform-34, build-tools 34)
aapt2 compile --dir app/src/main/res -o res.zip
aapt2 link -o base.apk -I $ANDROID_JAR --manifest app/src/main/AndroidManifest.xml --java gen res.zip
javac -source 8 -target 8 -bootclasspath $ANDROID_JAR -classpath gen:$BT/core-lambda-stubs.jar -d obj $(find gen app -name "*.java")
d8 --release --min-api 24 --lib $ANDROID_JAR --output apk $(find obj -name "*.class")
aapt add base.apk classes.dex
zipalign -f 4 base.apk out.apk
apksigner sign --ks debug.keystore --out final.apk out.apk
```

## 隐私声明

- 本项目不联网、不上传任何数据（INTERNET 权限仅为预留）
- 不读取通讯录、位置、存储等任何隐私
- 所有记录仅保存在应用私有目录

## License

MIT

