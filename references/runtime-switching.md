# 运行时接入：消费者与安全刷新

用于接入正文、Log、辅助窗口、菜单或其他用户指定消费者。这里的状态名与伪代码是设计示例，不是可直接注入任意游戏的 API。

## 先识别真正的数据流

沿“原语言资源 → 语句／历史身份 → 语言选择 → 样式与字体 → 布局缓存 → 可见窗口”追踪。正文可能读取消息对象的 locale，而 Log 读取 owner/context 的绑定或自己的过滤器；修改全局设置不保证已缓存的文字重绘。

记录每个消费者的原文来源、目标语言入口、字体入口、首次可绘制时机、全量／局部更新函数、关闭与销毁方式。区分：当前句、副本式预览、历史列表、名字等可能是不同窗口，即使外观相同。

官方 locale setter 也要验证作用域。用户只需字幕时，如果它同时改菜单、语音或持久设置，先确认有无受支持的局部上下文／消费者覆盖；不存在安全隔离方式时报告限制，不擅自扩大范围，也不反复全局切换后再强行恢复其他系统来掩盖副作用。

如果存档历史仅保存单语字符串，不能假定能从它还原另一语言。优先从场景 ID、语句 ID 或原资源解析；身份已丢失时说明旧历史的限制，必要时只支持新记录，不用相似度猜译文并覆盖存档。

## 保留原运行协议

- 核实实际编译器／VM、字节码格式与闭包上下文，匹配版本后才解析、编译或包装。不要因函数名类似就套用 JavaScript generator、Lua coroutine 或某种 VM 的惯例。
- 原循环可能在原生函数中暂停等待动画。包装层须保持调度、暂停／恢复、返回值、异常、参数、默认值、继承与闭包环境。某些 VM 的 `.call()` 会引入不可跨越的原生暂停边界；只有在该 VM 验证后，才采用绑定环境再直接脚本调用等替代路径。
- 测试应包含真正的 native suspend/resume 或等效真实调度，不用“同步空函数替身能返回”证明协程兼容。
- 确认上下文是 table、实例还是只读／锁定对象；不要向不能新增字段的实例写新槽。有些 VM 在实例化后锁定类，或子类持有父方法快照；补丁安装时机、继承分支都需核实。

## 最小状态模型

请求语言独立于窗口已呈现语言；首次 readiness 由官方绘制／生命周期证明，不由轮询猜测。窗口关闭再打开时清除旧 readiness 和完成标记；存在异步任务时使用实例世代／取消机制防止旧任务写回。

```text
onHotkeyEdge:
    requestedLocale = nextConfiguredLocale()
    requestRevision += 1               # 一个按下沿只消费一次

onOfficialFirstDrawCompleted(surface):
    surface.ready = true              # 不提前替官方创建原生资源

onSafeRefreshPoint(surface):
    if !surface.ready or surface.destroyed or surface.refreshing: return
    if surface.fullTextLocale != requestedLocale:
        request = captureLocaleRevisionAndInstanceGeneration()
        task = beginRefreshOwnedByInstanceAndGeneration(surface, request.generation)
        try:
            snapshot = capturePresentationState(surface)
            renderFullTextFromOriginalIdentity(surface, request.locale)
            waitForActualCompletionUsingEngineProtocol()
            if stillSameLiveInstance(surface, request.generation):
                restoreScrollAndFocus(surface, snapshot)
                surface.fullTextLocale = request.locale  # 确认实际画出的目标
        finally:
            releaseOnlyCapturedTaskState(task) # 不能清掉重开窗口／新任务的状态
        if stillSameLiveInstance(surface, request.generation) and request.revision != requestRevision:
            scheduleNextSafeRefresh(surface)   # 不误记为最新请求已经完成

onCursorOnlyOrCurrentLineRefresh(surface):
    refreshOnlyRequestedPart(surface)
    # 不确认整个历史列表已经呈现目标语言
```

按引擎选择同步提交、版本号或独立局部标记；不要求照抄这套字段。同步绘制无需额外等待；异步完成与再次按键／销毁重开并发时，只提交捕获的实际目标，过期任务不能操作新实例。实例／世代与任务归属校验须覆盖原生回调、滚动恢复和 finally 清理，不只最后的完成标记。刷新抛错时记录失败阶段、保留可恢复状态，不吞错后把语言标为已完成。只能安全逐帧更新时说明短暂等待；不能满足“当前句即时切换”时明确报告，而非改成下一句生效冒充完成。

## 字体、文字与布局

分别验证资源编码、VM 字符串、原生调用参数、渲染器接收值。沿用引擎原生编码或显式可逆转换；UTF-8／UTF-16／旧代码页都不是通用默认值。记录编码失败，不用 replacement character 或丢弃非法字节继续构建。

检查字体实际覆盖、回退链、样式作用域、标点／组合字符、换行和语言需要的 shaping／双向排版。字体可从官方语言配置追踪，但读取并不等于允许再分发。保留格式参数、富文本、控制码及语音／口型等非文本元数据，不通过整包跨版本覆盖改变它们。

缺译回退条目按实际显示的源语选择字形与排版；将“该请求已完成呈现”和“该条存在目标译文”分开，既不冒充译文覆盖，也不因永久缺译而每帧无限重绘。

切换后刷新必要的测量、布局、行高及裁剪缓存；验证长句完整可读及滚动末尾。通常只显示一个语言块，无需双语同屏挤压；也不能因单语切换就假定所有语言布局相同。

## 诊断顺序

按键事件到达 → 请求语言变化 → 正确消费者收到请求 → 原资源身份解析 → 安全绘制 → 字体／布局完成。每段用最小局部日志或可观察状态定位，诊断包装本身不得改变协程语义。

资源重打包后重新解析最终产物并比较允许改动的脚本／字段。只用匹配格式的解析器，工具退出码为零不代表实际文件已正确生成。原生挂接确有必要时限定职责、校验版本／签名，不把某个 DLL 名、函数地址、数组槽或平台做成通用假设；不绕过 DRM、反作弊或在线完整性保护。

