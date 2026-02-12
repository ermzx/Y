# 手写输入崩溃修复记录

> 修复日期: 2026-02-06
> 关联 Issue: [#756](https://github.com/gurecn/YuyanIme/issues/756) / [#687](https://github.com/gurecn/YuyanIme/issues/687)

---

## 问题概述

### 崩溃信息
```
java.lang.NullPointerException: Attempt to read from null array
    at os.onTouchEvent(SourceFile:248)
```

### 问题特征
- 刚安装/清除数据后手写正常
- 使用一段时间后，手写一笔抬手即闪退
- 清除应用数据可临时恢复，但丢失设置
- 影响系统: HarmonyOS 4.2.0

---

## 根本原因

### 直接原因
`HWEngine.kt:33` 对 native 引擎返回值的数组无边界检查：
```kotlin
val candidatesPyComposition = HandWriting.getCandidatesPyComposition()
val candidates = candidatesPyComposition[0]  // ← 数组为空时崩溃
```

### 间接原因
`HandWriting.init()` 使用外部存储目录存储手写数据：
```kotlin
val result = initWithDirectory(context, context.getExternalFilesDir("hw").toString())
```

长期使用后，native 引擎内部状态或缓存文件损坏，导致 `getCandidatesPyComposition()` 返回空数组。

---

## 修复内容

### 文件修改清单

| 文件 | 修改位置 | 修改内容 | 优先级 |
|------|---------|---------|--------|
| HWEngine.kt | 第 33 行 | `getOrNull(0) ?: return` | P0 |
| HWEngine.kt | 第 29-30 行 | 添加 `intArray.isEmpty()` 检查 | P0 |
| HWEngine.kt | 第 37 行 | `candidate!!` → null check | P0 |
| HandwritingKeyboard.kt | 第 90 行 | `mService!!` → `mService?` | P1 |
| HandwritingKeyboard.kt | 第 87,96,100 行 | `addPoint(getNewPoint(...))` → `?.let` | P1 |
| HandwritingKeyboard.kt | 第 135,138 行 | `tmp.c2!!/c1!!` → `?: return` | P1 |
| HandwritingKeyboard.kt | 第 132 行 | 参数类型 `TimedPoint?` → `TimedPoint` | P1 |
| HandwritingKeyboard.kt | 第 155 行 | `getNewPoint(...)!!` → `?.let` | P1 |
| HandwritingKeyboard.kt | 第 127 行 | `point!!` → `point?.let` | P1 |
| HandwritingKeyboard.kt | 第 186 行 | `canvas!!` → `canvas?` | P2 |
| HandwritingKeyboard.kt | 第 231 行 | `mSoftKeyboard!!` → `mSoftKeyboard?` | P2 |
| HandwritingKeyboard.kt | 第 241 行 | 回调中 `mService!!` → `mService?` | P2 |
| HandwritingKeyboard.kt | 第 121,224 行 | 其他 `!!` 解包 | P2 |

### 详细代码修改

#### 1. HWEngine.kt (主崩溃修复)

**文件**: `yuyansdk/src/main/java/com/yuyan/inputmethod/HWEngine.kt`

**修改前 (第 26-45 行)**:
```kotlin
fun recognitionData(strokes: MutableList<Short?>, recogResult: IHandWritingCallBack){
    HandWriting.reset()
    val strokesData = strokes.toMutableList()
    val intArray = strokesData.filterNotNull().map { it.toInt() }.toIntArray()
    HandWriting.inputHWPoints(intArray)
    val candidatesPyComposition = HandWriting.getCandidatesPyComposition()
    val candidates = candidatesPyComposition[0]  // ← 崩溃点
    if (candidates != null && candidates.isNotEmpty()) {
        val recogResultData = recogResult
        val recogResultItems = ArrayList<CandidateListItem>()
        for (candidate in candidates) {
            recogResultItems.add(
                CandidateListItem(
                    PinyinHelper.toHanYuPinyin(candidate!!, mHanyuPinyinOutputFormat, "'").ifEmpty { candidate }, candidate
                )
            )
        }
        recogResultData.onSucess(recogResultItems.toTypedArray())
    }
}
```

**修改后**:
```kotlin
fun recognitionData(strokes: MutableList<Short?>, recogResult: IHandWritingCallBack) {
    try {
        HandWriting.reset()
        val intArray = strokes.filterNotNull().map { it.toInt() }.toIntArray()
        if (intArray.isEmpty()) return
        HandWriting.inputHWPoints(intArray)
        val candidatesPyComposition = HandWriting.getCandidatesPyComposition()
        val candidates = candidatesPyComposition.getOrNull(0) ?: return
        if (candidates.isEmpty()) return
        val recogResultItems = ArrayList<CandidateListItem>()
        for (candidate in candidates) {
            if (candidate == null) continue
            recogResultItems.add(
                CandidateListItem(
                    PinyinHelper.toHanYuPinyin(candidate, mHanyuPinyinOutputFormat, "'")
                        .ifEmpty { candidate },
                    candidate
                )
            )
        }
        if (recogResultItems.isNotEmpty()) {
            recogResult.onSucess(recogResultItems.toTypedArray())
        }
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```

**关键改动**:
- `candidatesPyComposition[0]` → `candidatesPyComposition.getOrNull(0) ?: return` (第 33 行)
- `candidate!!` → `if (candidate == null) continue` (第 37 行)
- 添加 `intArray.isEmpty()` 前置检查 (第 30 行)
- 外层 try-catch 兜底

#### 2. HandwritingKeyboard.kt ( onTouchEvent )

**文件**: `yuyansdk/src/main/java/com/yuyan/imemodule/keyboard/HandwritingKeyboard.kt`

**修改后 (第 67-116 行)**:
```kotlin
@SuppressLint("ClickableViewAccessibility")
override fun onTouchEvent(me: MotionEvent): Boolean {
    if (!isEnabled || !InputModeSwitcherManager.isChineseHandWriting) {
        return super.onTouchEvent(me)
    }
    try {
        val eventX = me.x
        val eventY = me.y
        val softKey = onKeyPressHandwriting(eventX.toInt(), eventY.toInt())
        if (softKey != null) {
            return super.onTouchEvent(me)
        }
        mSBPoint.add(me.x.toInt().toShort())
        mSBPoint.add(me.y.toInt().toShort())
        when (me.action) {
            MotionEvent.ACTION_DOWN -> {
                val paintWidthMax = getInstance().handwriting.handWritingWidth.getValue() * 4 / 10f
                mMaxWidth = convertDpToPx(paintWidthMax)
                mMinWidth = convertDpToPx(paintWidthMax / 2f)
                parent.requestDisallowInterceptTouchEvent(true)
                mPoints.clear()
                getNewPoint(eventX, eventY)?.let { addPoint(it) }
                if (mLastUpTime != 0L && System.currentTimeMillis() - mLastUpTime > times) {
                    val key = SoftKey()
                    mService?.responseKeyEvent(key)
                    mSBPoint.clear()
                    clear()
                }
            }
            MotionEvent.ACTION_MOVE -> {
                getNewPoint(eventX, eventY)?.let { addPoint(it) }
                updatePathDelayed()
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                getNewPoint(eventX, eventY)?.let { addPoint(it) }
                parent.requestDisallowInterceptTouchEvent(true)
                mLastUpTime = System.currentTimeMillis()
                mSBPoint.add(-1)
                mSBPoint.add(0)
                recognitionData()
                updatePathDelayed()
            }
            else -> return false
        }
        invalidate()
        return true
    } catch (e: Exception) {
        e.printStackTrace()
        return false
    }
}
```

**关键改动**:
- `mService!!` → `mService?` (第 90 行)
- `addPoint(getNewPoint(...))` → `getNewPoint(...)?.let { addPoint(it) }` (第 87,96,100 行)
- 外层 try-catch

#### 3. HandwritingKeyboard.kt ( addPoint )

**修改后 (第 135-162 行)**:
```kotlin
private fun addPoint(newPoint: TimedPoint) {
    mPoints.add(newPoint)
    val pointsCount = mPoints.size
    if (pointsCount > 3) {
        var tmp = calculateCurveControlPoints(mPoints[0], mPoints[1], mPoints[2])
        val c2 = tmp.c2 ?: return
        recyclePoint(tmp.c1)
        tmp = calculateCurveControlPoints(mPoints[1], mPoints[2], mPoints[3])
        val c3 = tmp.c1 ?: return
        recyclePoint(tmp.c2)
        val curve = Bezier(mPoints[1], c2, c3, mPoints[2])
        // ... 贝塞尔曲线处理
        recyclePoint(mPoints.removeAt(0))
        recyclePoint(c2)
        recyclePoint(c3)
    } else if (pointsCount == 1) {
        val firstPoint = mPoints[0]
        getNewPoint(firstPoint.x, firstPoint.y)?.let { mPoints.add(it) }
    }
}
```

**关键改动**:
- 参数类型 `TimedPoint?` → `TimedPoint`
- `tmp.c2!!` → `tmp.c2 ?: return`
- `tmp.c1!!` → `tmp.c1 ?: return`
- `getNewPoint(...)!!` → `getNewPoint(...)?.let`

#### 4. 其他修复

**recyclePoint (第 131-133 行)**:
```kotlin
private fun recyclePoint(point: TimedPoint?) {
    point?.let { mPointsCache.add(it) }
}
```

**addBezier (第 186 行)**:
```kotlin
mSignatureBitmapCanvas?.drawPoint(x, y, mPaint)
```

**onKeyPressHandwriting (第 235-237 行)**:
```kotlin
private fun onKeyPressHandwriting(x: Int, y: Int): SoftKey? {
    return mSoftKeyboard?.mapToKey(x, y)
}
```

**recognitionData 回调 (第 239-243 行)**:
```kotlin
private fun recognitionData() {
    HWEngine.recognitionData(mSBPoint) { item ->
        mService?.postDelayed({ mService?.responseHandwritingResultEvent(item) }, 20)
    }
}
```

---

## 编译结果

### 环境

- **JDK**: OpenJDK 17.0.18
- **Gradle**: 8.7
- **Android SDK**: API 36
- **NDK**: 26.1.10909125
- **系统**: Linux (CachyOS)

### 生成的 APK

| 版本 | 文件名 | 大小 | 路径 |
|------|--------|------|------|
| Offline Debug | yuyanIme_2026020620_offline_debug.apk | 64MB | app/build/outputs/apk/offline/debug/ |
| Online Debug | yuyanIme_2026020620_online_debug.apk | 64MB | app/build/outputs/apk/online/debug/ |

### 编译命令

```bash
# 配置环境
export ANDROID_HOME=/home/eric/Android/sdk
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools

# 创建签名密钥
keystore/keystore.properties

# 编译 Debug APK
./gradlew assembleDebug
```

---

## 验证建议

### 功能测试

1. **基本功能**: 手写输入单字、连续书写多字
2. **长时间使用**: 持续使用 30 分钟以上
3. **边界条件**: 极快书写、单点触摸、多指触摸
4. **生命周期**: 切换输入模式、应用切后台再恢复
5. **异常恢复**: 手动删除 `hw/` 目录后使用

### 测试步骤

1. 安装 APK 到真实设备
2. 切换到手写输入模式
3. 连续手写输入 50+ 个汉字
4. 等待 10 分钟后再手写测试
5. 切换到其他应用再返回，继续手写

---

## 相关 Issue

- Issue #756: 20251222.16版本手写输入还是闪退
- Issue #687: 手写输入时不上屏

---

修复完成日期: 2026-02-06
修复人员: opencode AI

---

# 后续问题修复方案建议（候选词消失但不崩溃）

> 记录日期: 2026-02-08
> 目标: 解决“使用一段时间后手写无候选词但不崩溃”的问题

## 现象复盘

- 经过 NPE 修复后，崩溃消失
- 但在连续使用一段时间后，手写输入不再产生候选词
- 清除应用数据/缓存可临时恢复

## 根因分析（更高置信度）

`mSBPoint`（手写点缓存）在一次识别完成后并不会清空，仅在“下一次按下且间隔超过 times”时才清空。

这会导致：

- 旧笔迹持续累积在内存中
- 超长/过期笔迹被重复喂给 native 引擎
- native 引擎返回空候选数组
- Java 层现在安全 return → 表现为“无候选词”

**相关位置**:

- 手写点累积：`HandwritingKeyboard.kt` onTouchEvent 中的 `mSBPoint.add(...)`
- 仅在超时分支清理：`if (System.currentTimeMillis() - mLastUpTime > times) { mSBPoint.clear() }`
- 识别完成不清理：`recognitionData()` 之后没有 reset

## 推荐修复方案（最小风险）

### 核心策略

在一次识别周期结束时**立即清空旧笔迹**，避免历史点进入下一次识别。

### 具体做法（优先级高）

1. **在识别调用后清空**
   - `recognitionData()` 触发后立即 `mSBPoint.clear()`
   - 可选：也在 `ACTION_CANCEL` 分支中清空，防止取消事件遗留

2. **设定最大笔迹点数上限**
   - 若 `mSBPoint.size` 超过阈值（如 4096/8192），自动清空并重置
   - 防止异常输入造成极端累积

3. **周期性清理保护**
   - 当 `times` 超时触发时，除了 `clear()` 画布，也清空 `mSBPoint`

### 参考实现（伪代码）

```kotlin
private fun recognitionData() {
    HWEngine.recognitionData(mSBPoint) { item ->
        mService?.postDelayed({ mService?.responseHandwritingResultEvent(item) }, 20)
    }
    mSBPoint.clear()
}

// ACTION_CANCEL 分支
mSBPoint.clear()

// 超时重置分支
if (System.currentTimeMillis() - mLastUpTime > times) {
    ...
    mSBPoint.clear()
    clear()
}

// 防护上限
if (mSBPoint.size > 8192) {
    mSBPoint.clear()
    clear()
}
```

## 方案优势

- 不依赖 native 内部状态是否健康
- 逻辑简单，回滚风险低
- 与“清除缓存能恢复”的表现一致

## 验证建议

1. 连续手写 10 分钟以上是否仍稳定出候选
2. 极快输入/多指输入是否导致异常候选中断
3. 触发超时清理后是否可正常继续识别

---

# 实施记录（已执行）

> 执行日期: 2026-02-08

## 修改清单

- `yuyansdk/src/main/java/com/yuyan/imemodule/keyboard/HandwritingKeyboard.kt`
  - 新增手写点缓存上限保护（`maxStrokePoints`），超过阈值即清空并重置画布
  - 识别完成后立即清空 `mSBPoint`，避免旧笔迹进入下一轮识别
  - 超时重置分支使用统一的 `clearStrokeBuffer()` 清理笔迹缓存

## 变更摘要

核心修复是**让一次识别周期只消费当前笔迹**，彻底切断历史点累积的路径。

---

# 意外问题与修复（连续性与边缘识别）

> 记录日期: 2026-02-08

## 发现的问题

1. **连续性极差（多笔画被截断）**  
   在 `recognitionData()` 之后立即 `clearStrokeBuffer()`，导致每次抬手就清空历史笔迹，无法累计多笔画。

2. **写到键盘边缘无候选**  
   `onTouchEvent()` 每个事件都会检测 `onKeyPressHandwriting()`，手指经过右侧功能键区域时被当作按键，后续点位被直接丢弃，导致候选为空或断笔。

## 根因定位

- 过度清理：`HandwritingKeyboard.kt` 的 `recognitionData()` 末尾清空 `mSBPoint`
- 过度拦截：`onKeyPressHandwriting()` 在 `ACTION_MOVE/UP` 也生效

## 修复策略

- **只在 `ACTION_DOWN` 判断是否点击功能键**；一旦判定为手写手势，移动过程中不再拦截键位
- **移除识别后立即清理**，让多笔画在 `times` 超时或新字开始时再清理

## 修复实现要点

- 增加 `isHandwritingGesture` 标志位
- `ACTION_DOWN` 若为功能键则交由 `super`，否则进入手写模式
- `ACTION_MOVE/UP` 仅在手写模式下采点

---

# TODO（潜在风险跟踪）

1. **多笔画合并策略默认值需要验证**  
   已加入“多笔画合并时间”设置项，需验证默认值对连续性与误分字的平衡。

2. **笔迹缓存上限需要验证合理阈值**  
   已加入“笔迹缓存上限”设置项，需在真实设备上验证默认值是否过大/过小。

3. **边缘功能键命中误判回归测试**  
   需要覆盖“靠近功能键区书写”的手动测试，确保不会再被误判为按键。

---

# 设置项落地（已实现）

> 执行日期: 2026-02-08

新增手写设置项：

1. **多笔画合并时间**  
   - key: `hand_writing_merge_timeout`  
   - 0 表示跟随“识别灵敏度”的默认逻辑  
2. **笔迹缓存上限**  
   - key: `hand_writing_max_stroke_points`  
   - 默认 8192 点  

对应实现位置：

- `yuyansdk/src/main/java/com/yuyan/imemodule/prefs/AppPrefs.kt`
- `yuyansdk/src/main/java/com/yuyan/imemodule/keyboard/HandwritingKeyboard.kt`
- `yuyansdk/src/main/res/values/strings.xml`

---

# 未解决问题（待进一步诊断）

> 记录日期: 2026-02-12

## 现象

用户反馈“问题仍然如常”，表现为：

- 连续性仍差（多笔画可能被截断）
- 边缘书写仍可能无候选

## 结论

当前改动未完全解决问题，**需要进一步诊断**。

## 下一步建议（诊断计划）

1. 收集日志：确认是否触发了“超时清理”或“功能键误判”路径  
2. 记录设置值：`hand_writing_merge_timeout`、`hand_writing_max_stroke_points`  
3. 确认复现步骤：抬手间隔时长、是否靠近右侧功能键区域

## 新反馈补充

- 用户反馈：故障出现后，将“笔迹缓存上限”调至最小值**仍无任何改善**。

## 推测

这说明问题**不只是内存点数累积**，更可能是：

- native 引擎内部状态异常（reset 无法恢复）
- 手写缓存文件损坏（需要重新初始化引擎或清理 hw 目录）

---

# 日志开关（已实现）

> 执行日期: 2026-02-12

新增手写调试日志开关：

- 设置项：**手写设置 → 手写调试日志**
- key: `hand_writing_debug_log`

记录内容（仅在开关开启时）：

- 手写开始/结束、点数统计
- 超时清理触发原因
- 候选词回调数量
- 引擎返回空候选的情况

建议抓取日志命令（参考）：

```bash
adb logcat | grep -E "HandwritingKeyboard|HWEngine"
```

---

# 自愈机制（已实现）

> 执行日期: 2026-02-12

当引擎连续多次返回空候选（且点数足够）时，会自动尝试 **reinit 引擎**：

- 不清理用户配置
- 默认不删除 `hw/` 缓存目录（仅重启引擎）
- 连续空候选达到阈值后触发（当前为 3 次）
