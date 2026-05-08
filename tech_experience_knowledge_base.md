# 【合并】用户全部技术开发经验与方法知识库（综合版）

> 本条记忆由 Operit 在 2026-05-08 从约58条开发/技术/方法相关记忆中合并提炼而成，删除重复冗余，保留全部核心经验。

---

## 📌 核心方法论

### 首要原则：不要闭门造车，先查资料再动手
- **做任何项目前**，先查已有方案、先看官方文档、先找人踩过的坑
- **遇到错误**，先搜索错误信息而不是自己瞎试
- **拿不准的代码**，先查源码/API文档再写

### AI操作守则：必须主动推理，而非机械执行
- 用户极其重视推理能力，不能机械套用已有资料
- 每步操作前先想清楚：当前在哪里、目标在哪、该走什么路径
- 数据获取不完整时合理推断，总结时要阐述推理逻辑

---

## 🚀 项目一：虚拟定位APP

### 项目概况
- 包名：`com.virtuallocation.app`
- 技术栈：Kotlin + Jetpack Compose + Material3 + Shizuku
- 版本：v5.4（最终版，versionCode=9）
- 工作区：`6c9c9b2f-2e5b-432b-9312-d599f08212a6`
- APK路径：`/sdcard/Download/virtuallocation_v5.4.apk`
- 项目备份：`/sdcard/Download/virtuallocation_project_final.zip`
- 编译方式：`gradle assembleDebug --no-daemon > build.log 2>&1`

### 核心架构
- **MockLocationHelper.kt / MockLocationService.kt**：核心引擎，协程/Handler持续推送gps/network/passive三个源
- **MainActivity.kt**：Compose UI，坐标输入、收藏功能、启动/停止
- **AdbCommandActivity.kt**：ADB广播接收，支持传action参数
- **ShizukuShell.java**：反射调用Shizuku执行shell命令
- **MockQSTileService.kt**：快速设置磁贴（TileService）

### ADB启动命令（最终可用版）
```bash
# 启动收藏项
am start -n com.virtuallocation.app/.AdbCommandActivity --es action favorite --es name "泰捷"
# 直接传坐标启动
am start -n com.virtuallocation.app/.AdbCommandActivity --es action start --ef lat 24.4667 --ef lng 118.1167
```

### 全部踩坑记录

#### 1. testOnly="true" → 安装报-15
解决：普通APP + ACCESS_MOCK_LOCATION权限（加tools:ignore）

#### 2. 定位只发3次 → APP读不到
解决：协程每100-200ms持续推送，不要停止

#### 3. 只覆盖GPS → 部分APP不生效
解决：同时覆盖 gps/network/passive 三个源

#### 4. ShizukuShell反射参数包装错误（最隐蔽的bug）
❌ `Method.invoke(null, new Object[]{new String[]{"sh"}, null, null})`
✅ `Method.invoke(null, new String[]{"sh"}, null, null)`
**核心教训**：`Method.invoke(Object obj, Object... args)` 是可变参数，多包一层 `new Object[]{}` 导致参数结构错乱

#### 5. scheduleRetryRegister递归不停止
问题：registerProviders成功后没有return，递归继续调用，provider反复removed/added
修复：加入 `return@synchronized` 和 `if (running) return@synchronized` 双重保护

#### 6. startPushLoop中主线程sleep阻塞
问题：`Thread.sleep(100)` 在 `Handler.postDelayed` 回调中阻塞主线程
修复：改为Handler轮询burst快速连推（前3次150ms间隔，之后200ms），全程不阻塞

#### 7. 自动授权时序问题
问题：Activity.onCreate中调用grantMockLocation()无效，Shizuku Binder未就绪
修复：移到Service.onStartCommand开头直接try-catch执行

#### 8. 推送参数优化（提高APP采纳率）
- accuracy：8~20米随机
- speed：0.5~2.0随机
- 抖动：±10米随机偏移
- 推送前调用 setTestProviderStatus(AVAILABLE)

#### 诊断命令
```bash
dumpsys location | grep -A5 "gps provider \[mock\]"
dumpsys location | grep -E "last location=Location\[gps" | head -3
```

#### 已知无解
- 高德地图定位不生效（高德SDK主动检测isFromMockProvider）

---

## 🚀 项目二：万能APP（PermOpener继承）

### 项目概况
- 包名：`com.wanneng.app`（原 `com.java.myapplication`）
- 技术栈：Kotlin + Jetpack Compose + Shizuku + 无障碍服务
- 版本：v2.0
- 工作区：`162d58ed-a2d9-4a38-82c7-b418cbd8cd5c`
- APK路径：`/sdcard/Download/万能_v2.apk`

### 功能清单
1. **画框模式**：绿色网格覆盖层（60px间距），全穿透不拦截操作，点击闪光反馈（红圈300ms），音量+切换
2. **行为记录**：监听TYPE_VIEW_CLICKED记录text/viewId/className，监听TYPE_WINDOW_STATE_CHANGED记录页面切换
3. **权限全开**：pm grant 30项 + appops set 146项 + settings，继承自PermOpener
4. **音量键控制**：音量+切换画框，音量-切换行为记录

### 核心踩坑：无障碍服务+OVERLAY导致手机变砖

#### 致命组合拳
**无障碍服务（AccessibilityService） + 前台悬浮窗覆盖层 + START_STICKY + 高频事件监听 = 手机变砖**

#### 完整死循环链路
```
手机开机 → 系统自动启动无障碍服务 → onAccessibilityEvent高频触发
→ scanAndDraw不断扫描UI树 → OverlayDrawService叠加全屏覆盖层
→ 用户触摸无反应 → 重启→再次进入死循环
```

#### ✅ ADB急救方案
```bash
# 清空无障碍服务列表（最有效）
adb shell settings put secure enabled_accessibility_services ""
# 停用无障碍功能
adb shell settings put secure accessibility_enabled 0
# 安全模式启动后卸载
adb reboot safe-mode
```

#### ✅ 安全开发实践
- `onServiceConnected()` 不自动启动任何功能
- 用 `START_NOT_STICKY` 而非 `START_STICKY`
- 不监听 `TYPE_WINDOW_CONTENT_CHANGED`（频率极高）
- 所有功能由用户手动触发
- 开发测试用备用机，先连ADB

#### 万能APP代码结构错乱导致闪退（重要教训）
**问题**：多次`apply_file`替换后，`saveCheckedApps`函数被嵌套进别的函数体，变成局部函数；花括号数量不匹配（{52个，}53个）
**诊断**：
```bash
grep -o '{' MainActivity.kt | wc -l
grep -o '}' MainActivity.kt | wc -l
grep -n "private fun" MainActivity.kt
```
**教训**：`apply_file`大段替换容易导致嵌套错误；花括号必须匹配；运行时闪退看 `logcat -b crash -d`

---

## 🚀 项目三：财经新闻APP

### 项目概况
- 技术栈：Kotlin + Jetpack Compose + OkHttp + Gson（替代Room）
- 编译环境：AGP 9.0 + Kotlin 2.3（不能用kapt/ksp/annotationProcessor）

### 全部踩坑速查表
```
Cannot find ..._Impl does not exist → 删Room，换Gson+JSON文件
not compatible with built-in Kotlin support → 不用KAPT/KSP
No value passed for parameter 'value' → Kotlin 2.3强制初始值
collectLatest嵌套collect死锁 → 改用applyFilter()过滤模式
Cannot infer type for type parameter → 属性监听+手动过滤
INSTALL_FAILED_ABORTED → 从/data/local/tmp/安装
Syntax error in try-catch → 不用withTimeout，OkHttp超时就够
```

### 最佳实践模板
**1. JsonStorage替代Room**（filesDir存储JSON）
**2. SafeViewModel**：永远不要用collectLatest嵌套collect，只用修改状态→调用applyFilter()
**3. OkHttp在协程中**：直接suspend里调execute()（阻塞但OK），在async{}里调它

### Shizuku安装命令
```bash
cp /sdcard/Download/app.apk /data/local/tmp/
pm install -r /data/local/tmp/app.apk
```

---

## 🚀 项目四：彭博社新闻采集展示系统

### 项目概况
- 数据采集：Python + Playwright (async API)
- 数据解析：`page.inner_text('body')` 按行解析，三行一组（标题→相对时间→分类）
- 展示：自包含HTML+CSS+JS，数据嵌入 `const articles = [...]`
- HTTP服务：Python `http.server` 端口8899，绑定127.0.0.1
- 浏览器：X浏览器（`am start` 打开）

### 核心工作流
```
1. page.goto("https://m.bbwc.cn/") 获取body文本
2. 正则匹配 /^(\d+)\s*(小时|天|周)\s*前$/ → 向上2行取标题
3. document.querySelectorAll('a[href*="/article/"]') 匹配URL
4. page.goto(url) → evaluate多选择器兜底取正文前3000字
5. 生成HTML注入数据 → 启动HTTP → 浏览器打开
```

### 关键经验
- 不用DOM选择器，用文本匹配更稳定
- 日期从相对时间取（"2小时前"），不从栏目日期取
- 链接匹配用 `includes()` 而非 `===`
- 详情页正文8个选择器降级：article→.article-content→.content→main→p集合
- Android 11+ 不能用 file://，必须HTTP服务

---

## 🚀 项目五：FloatRunner 浮窗命令执行器

- 技术栈：Java + Shizuku
- 定位：极简浮窗版Shizuku，只有坐标输入框+启动/收藏
- 版本：v1.0，单文件，无数据库，SharedPreferences持久化

---

## 🚀 项目六：Shizuku AI（AI悬浮窗助手）

### 项目概况
- 包名：`com.shizuku.ai`
- 技术栈：Java + Shizuku API 13.1.5 + DeepSeek API + WindowManager悬浮窗
- 版本：v1.4.0
- 工作区：`53c8fe76-b9e5-4e56-9eca-8fb6c4b190bf`
- APK路径：`/sdcard/Download/shizuku_ai_v1.4.0.apk`

### 功能清单
1. AI模式：用户说人话需求，AI理解并执行adb命令
2. 200+条内置ADB命令库（11大分类），分词搜索
3. AI动态学习新命令（[LEARN]标签自动保存）
4. AI给命令加备注（[NOTE]标签）和标签（[TAG]标签）
5. 联网搜索（DuckDuckGo lite，无需API Key）
6. Token持久化保存，每次启动清空上次对话
7. 叉子彻底关闭（停掉所有API线程）
8. 拖动下限限制 + 长按清除按钮重置位置
9. 连续3次相同命令自动拦截防循环
10. **list_apps显示中文名**：shell拿包名 + getApplicationInfo(包名).loadLabel(pm)
11. **启动应用**：monkey -p 包名 1（替代am start -n）
12. **悬浮窗权限自动授权**：appops set包名 SYSTEM_ALERT_WINDOW allow

### 核心经验
- 悬浮窗权限用appops set通过Shizuku授权
- ShizukuShell反射参数不能多包一层
- 编译用assembleDebug，Release被lint拦截
- am force-stop输出为空=成功

---

## 🛠️ 安卓项目构建通用方案

### gradle.properties加速配置
```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
kotlin.daemon.jvmargs=-Xmx2048m
kotlin.incremental=true
kotlin.compiler.execution.strategy=in-process
android.useAndroidX=true
android.nonTransitiveRClass=true
```

### 国内镜像源（settings.gradle.kts）
配置阿里云+华为云镜像加速依赖下载

### 构建耗时（移动端环境）
- 首次全量下载依赖：~12分钟
- 配置缓存首次生成：~5.5分钟
- 配置缓存复用+全部缓存命中：~10秒
- 改代码后：~30秒~2分钟

### 关键命令
```bash
./gradlew assembleDebug — 构建Debug APK
./gradlew installDebug — 构建并安装
export ANDROID_HOME=/root/Android — SDK路径
AAPT2：使用linux-aarch64二进制文件
```

---

## 📱 AutoJS6 UI自动化操作手册

### 核心原则
**能用shell直连不要启动AutoJS6**，避免界面被遮挡和额外开销。

### 脚本文件
1. **读取ui.js**：`/sdcard/脚本/Operit/读取ui.js` → 抓取当前屏幕控件 → current_ui.json
2. **点击目标坐标.js**：`/sdcard/脚本/Operit/点击目标坐标.js` → 读取目标坐标.txt → shizuku('input tap x y')
3. **目标坐标.txt**：`/sdcard/脚本/Operit/目标坐标.txt` → 存储坐标 x,y

### 运行命令
```bash
am start -n org.autojs.autojs6/...RunIntentActivity -a android.intent.action.VIEW \
  -d /storage/emulated/0/脚本/Operit/<脚本名.js> -f 0x10000000
```

### 标准流程
1. `am start` 打开目标APP
2. 运行读取ui.js → 查看current_ui.json找目标坐标
3. `echo -n "x,y" > 目标坐标.txt`
4. 运行点击目标坐标.js
5. 再次运行读取ui.js确认页面变化

### 踩坑
- ❌ ForegroundService启动 → 后台无toast，点击无效
- ❌ click()或class name点击 → 支付宝等有触摸防护
- ✅ 必须用 `shizuku('input tap x y')` 绕过触摸防护
- ✅ 同一文字在不同页面区域含义不同，必须结合坐标判断页面模式

### 页面特征快速判断
- **首页**：有"扫一扫""收付款""出行"等快捷卡片
- **我的页**：出现用户昵称、"总资产""账单"等列表项
- **总资产页**：出现"活期资产""稳健理财""进阶理财"分区

### 浏览器搜索通用方案（省Token版）
```bash
am start -a android.intent.action.VIEW -d "https://www.baidu.com/s?wd=关键词"
# 验证
dumpsys window | grep mCurrentFocus
# → com.mmbox.xbrowser/.BrowserActivity 即为成功
```

---

## 📊 金融数据接口库测试结论

| 库 | 评分 | 说明 |
|---|------|------|
| **akshare** | ✅ 首推 | 东方财富tushare新闻API最新可用，token不存在也能用 |
| tushare | ⚠️ 需要Token | 注册拿Token后可用 |
| efinance | ❌ 不推荐 | 仅支持股票列表，不支持新闻 |

### akshare跑通步骤
```python
import akshare as ak
df = ak.news_dongfangcaifu_tushare()  # 无需token
df = ak.stock_zh_a_hist(symbol="600000", period="daily")
```

---

## 🖥️ Playwright浏览器自动化经验

### 页面解析技巧
- `page.inner_text('body')` 文本解析比DOM选择器更稳定（页面结构变化时仍可用）
- 链接匹配用 `includes()` 而非 `===`（标题被截断时）
- 正文提取8选择器降级：article→.article-content→.content→main→p集合

### 彭博社新闻采集
- **文本采集**：goto iBloomberg → inner_text('body') → 按行拆分
- **时间匹配**：`/^(\d+)\s*(小时|天|周)\s*前$/`
- **标题提取**：时间匹配行向上2行取标题
- **链接匹配**：`a[href*="/article/"]` 标题包含匹配

---

## ⚠️ GitHub上传经验

### 认证信息
- 用户名：lcydtc15967073371
- 经典Token（git push用）：TOKEN_REMOVED
- 邮箱：EMAIL_REDACTED

### 核心教训：永远从工作区复制源码，不从zip备份
✅ 来源：`/data/user/0/com.ai.assistance.operit/files/workspace/{workspaceId}/`
❌ 避免：`/sdcard/Download/` 下的zip备份（可能不是最新版）

### 上传流程
```bash
# 1. copy_file从Android工作区复制到Linux
# 2. 清理 .backup/ .operit/ app/build/ .gradle/ tools/
# 3. git init → git add → git commit
# 4. git remote add origin https://用户名:Token@github.com/用户/仓库.git
# 5. git push -f -u origin main
```

### 工作区路径速查
| 项目 | 工作区ID |
|------|---------|
| 虚拟定位APP v5.4 | `6c9c9b2f-2e5b-432b-9312-d599f08212a6` |
| FloatRunner v1.0 | 不在工作区（直接clone） |
| PermOpener v3.3 / 万能APP | `162d58ed-a2d9-4a38-82c7-b418cbd8cd5c` |
| Shizuku AI v1.4.0 | `53c8fe76-b9e5-4e56-9eca-8fb6c4b190bf` |
| 财经新闻APP | 同万能APP工作区 |

### APK仓库
- 仓库：`lcydtc15967073371/apk_files`
- 当前内容：FloatRunner_1.0.apk(886KB) + 虚拟定位_5.4.apk(12MB) + shizuku_ai_v1.2.0.apk(997KB)

### 踩坑记录
1. curl检查仓库时不要混注释：`# 注释` 和curl分开执行
2. git push报Authentication failed → 检查 `git remote -v`
3. git push报Recv failure → commit混入大文件，清理后再重试
4. git push报权限问题 → 重新 `git remote set-url origin https://用户名:Token@...`

---

## 📲 ADB/Shizuku通用经验

### 打开APP通用流程
```bash
# 1. 查包名
pm list packages | grep 关键词
# 2. 查启动Activity
dumpsys package 包名 | grep -A 2 "android.intent.action.MAIN"
# 3. 打开
am start -n 包名/.Activity名
```

### 已测成功的APP
- **同花顺**：com.hexin.plat.android/.LogoEmptyActivity
- **X浏览器**：com.mmbox.xbrowser/.BrowserActivity
- **支付宝**：com.eg.android.AlipayGphone/.AlipayLogin

### ShizukuShell核心经验
- 反射调用`Method.invoke(null, args)`不要多包`new Object[]{}`
- `exec()`方法执行前先try-catch，不要检查isAvailable/isGranted
- 服务中（Service.onStartCommand）比Activity中更可靠
- 异常不要完全吞掉，最少输出到logcat
- 主线程不要sleep，用Handler.post替代
- 递归函数要有明确终止条件

### 无障碍服务开发安全守则
1. 永远在备用机上测试，先连ADB
2. 不监听TYPE_WINDOW_CONTENT_CHANGED（频率极高）
3. `onServiceConnected()`不自动启动任何功能
4. 用 `START_NOT_STICKY` 而非 `START_STICKY`
5. 所有功能由用户手动触发
6. 准备好ADB急救命令：`settings put secure enabled_accessibility_services ""`
7. 花括号数量必须匹配，`grep -o '{' file | wc -l` 和 `grep -o '}' file | wc -l` 养成习惯
8. `apply_file`大段替换容易出结构问题，优先完整重写

### list_installed_apps读取原理
- shell拿包名：`pm list packages -3`
- Java侧查中文名：`getApplicationInfo(包名, 0).loadLabel(pm)`
- 关键词过滤在Java侧做
- `getApplicationInfo(指定包名)` 是精准查询，不受Android 11包可见性限制

---

## 🧠 用户技术栈画像

- **量化交易偏执狂**：自建网格交易系统，AutoJs6脚本盯盘，网格3.3%/倍数1.38/叠加买入分叉
- **银行股死多头**：7只银行股，大部分盈利30%+
- **搞机老手**：Shizuku+黑阈+冰箱+权限狗全链路通
- **技术驱动型**：DeepSeek API自搭前端，AutoJs6自研17+脚本
- **隐私敏感**：微多开+VPN+虚拟定位+权限管理全覆盖
- **AI全平台用户**：DeepSeek/豆包/元宝/千问/Gemini/Operit全用
- 所在地：浙江温岭，自由职业，与制造/贸易/投资相关
