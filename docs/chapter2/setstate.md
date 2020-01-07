> 本文来自整理和简化

调用 setState()必须是没有调用过 dispose()方法，不然出错，可通过` mounted `属性来判断调用此方法是否合法。
```dart
if (mounted) {
  setState(() {});
}
```

清晰的看到在framework.dart内setstate方法除了一些条件判断就是：
```dart
_element.markNeedsBuild();
```
那我们看看markNeedsBuild。
## Element 类 markNeedsBuild方法
```dart
  void markNeedsBuild() {
    assert(_debugLifecycleState != _ElementLifecycle.defunct);
    if (!_active)
      return;//返回
     ...
    if (dirty)
      return;
    _dirty = true;
    //调用scheduleBuildFor方法
    owner.scheduleBuildFor(this);
  }

```
将 element 元素标记为“脏”,并把它添加到全局的“脏”链表里,以便在下一帧更新信号时更新.
* 这里的“ `脏`”链表是待更新的链表，更新过后就不“脏”了。
* 由于一帧做两次更新有点低效，所以在`_active=false` 的时候直接返回。

那我们看看本方法最后调用的scheduleBuildFor方法。
## BuildOwner 类 scheduleBuildFor方法
`BuildOwner`类是`widget framework`的管理类，它跟踪那些需要重新构建的 widget。

```dart
void scheduleBuildFor(Element element) {
     ...
    if (element._inDirtyList) {
      ...
      _dirtyElementsNeedsResorting = true;
      return;
    }
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled();//回调
    }
    _dirtyElements.add(element);//把element加入脏元素链表
    element._inDirtyList = true;
    assert(() {
      if (debugPrintScheduleBuildForStacks)
        debugPrint('...dirty list is now: $_dirtyElements');
      return true;
    }());
  }
  ```
把一个 element 添加到 _dirtyElements 链表，以便当`WidgetsBinding.drawFrame`中调用 buildScope 的时候能够重构 element。onBuildScheduled()是一个 BuildOwner 的回调。

onBuildScheduled回调在WidgetsBinding的initInstances里初始化。
```dart
mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    // 这里
    buildOwner.onBuildScheduled = _handleBuildScheduled;
    window.onLocaleChanged = handleLocaleChanged;window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
 SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
SystemChannels.system.setMessageHandler(_handleSystemMessage);  FlutterErrorDetails.propertiesTransformers.add(transformDebugCreator);
  }
}
```
我们可以看到buildOwner.onBuildScheduled回调等于了_handleBuildScheduled，那现在来看看这个_handleBuildScheduled方法：
```dart
void _handleBuildScheduled() {
    //调用ensureVisualUpdate
    ensureVisualUpdate();
  }
```
可以看到调用ensureVisualUpdate方法，那我们继续走下去。

# SchedulerBinding类ensureVisualUpdate方法
```dart
  void ensureVisualUpdate() {
    switch (schedulerPhase) {
      case SchedulerPhase.idle:
      case SchedulerPhase.postFrameCallbacks:
        //当schedulerPhase为SchedulerPhase.idle，
        //SchedulerPhase.postFrameCallbacks时调用
        //scheduleFrame()
        scheduleFrame();
        return;
      case SchedulerPhase.transientCallbacks:
      case SchedulerPhase.midFrameMicrotasks:
      case SchedulerPhase.persistentCallbacks:
        return;
    }
  }
```
分别case了SchedulerPhase 的 5 个枚举值：
状态|含义
--|:--:
idle|没有正在处理的帧，可能正在执行的是 WidgetsBinding.scheduleTask，scheduleMicrotask，Timer，事件 handlers，或者其他回调等
transientCallbacks|SchedulerBinding.handleBeginFrame 过程， 处理动画状态更新
midFrameMicrotasks|处理 transientCallbacks 阶段触发的微任务（Microtasks）
persistentCallbacks|WidgetsBinding.drawFrame 和 SchedulerBinding.handleDrawFrame 过程，build/layout/paint 流水线工作
postFrameCallbacks|主要是清理和计划执行下一帧的工作

# 第二个case调用scheduleFrame()方法
那我们看看scheduleFrame()方法
```dart
  void scheduleFrame() {
  if (_hasScheduledFrame || !_framesEnabled) return;
  assert(() {
    if (debugPrintScheduleFrameStacks)
      debugPrintStack(
          label: 'scheduleFrame() called. Current phase is $schedulerPhase.');
    return true;
  }());
  //调用Window 的scheduleFrame方法是一个 native 方法
  window.scheduleFrame();
  _hasScheduledFrame = true;
}
  ```
WidgetsFlutterBinding 混入的这些 Binding 中基本都是监听并处理 Window 对象的一些事件，然后将这些事件按照 Framework 的模型包装、抽象然后分发。可以看到 WidgetsFlutterBinding 正是粘连 Flutter engine 与上层 Framework 的“胶水”。
|名|解释
--|:--:
GestureBinding|提供了 window.onPointerDataPacket 回调，绑定 Framework 手势子系统，是 Framework 事件模型与底层事件的绑定入口
ServicesBinding|提供了 window.onPlatformMessage 回调， 用于绑定平台消息通道（message channel），主要处理原生和 Flutter 通信
SchedulerBinding|提供了 window.onBeginFrame 和 window.onDrawFrame 回调，监听刷新事件，绑定 Framework 绘制调度子系统
PaintingBinding|绑定绘制库，主要用于处理图片缓存
SemanticsBinding|语义化层与 Flutter engine 的桥梁，主要是辅助功能的底层支持
RendererBinding|提供了 window.onMetricsChanged 、window.onTextScaleFactorChanged 等回调。它是渲染树与 Flutter engine 的桥梁
WidgetsBinding|提供了 window.onLocaleChanged、onBuildScheduled 等回调。它是 Flutter widget 层与 engine 的桥梁

之前的文中有说过，UI 的绘制逻辑是在 Render 树中实现的，所以这里还来细看 RendererBinding 的逻辑。

# RendererBinding
```dart
void initInstances() {
  ...

  //监听Window对象的事件
  ui.window
    ..onMetricsChanged = handleMetricsChanged
    ..onTextScaleFactorChanged = handleTextScaleFactorChanged
    ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
    ..onSemanticsAction = _handleSemanticsAction;

  //添加PersistentFrameCallback
  addPersistentFrameCallback(_handlePersistentFrameCallback);
}
```
addPersistentFrameCallback 中添加 _handlePersistentFrameCallback 最终调用了 drawFrame 而 WidgetsBinding 重写了 RendererBinding 中的 drawFrame() 方法。最终发现我们又回到了 WidgetsBinding 这个类中，在 WidgetsBinding 中 drawFrame 的实现如下：
```dart
@override
void drawFrame() {
 ...
  try {
    if (renderViewElement != null)
      // 重构需要更新的element
      buildOwner.buildScope(renderViewElement);
    super.drawFrame(); //调用RendererBinding的drawFrame()方法
    buildOwner.finalizeTree();
  }
}
```
在上面 scheduleBuildFor 方法介绍中有提到："scheduleBuildFor 是把一个 element 添加到 _dirtyElements 链表，以便当[WidgetsBinding.drawFrame]中调用 buildScope 的时候能够重构 element。onBuildScheduled()是一个 BuildOwner 的回调"。在 drawFrame 中调用 buildOwner.buildScope(renderViewElement)更新 elements。
```dart
  void buildScope(Element context, [ VoidCallback callback ]) {
    ...
      while (index < dirtyCount) {
        assert(_dirtyElements[index] != null);
        assert(_dirtyElements[index]._inDirtyList);
        assert(!_dirtyElements[index]._active || _dirtyElements[index]._debugIsInScope(context));
        try {
          //while 循环进行元素重构
          _dirtyElements[index].rebuild();
        } catch (e, stack) {
        ...
        }
      }
  }

```

# 得出

### 条件判断

- 1.生命周期判断
- 2.是否安装mounted

### 管理类

- 1.告诉管理类方法自己需要被重新构建
- 也就是BuildOwner类scheduleBuildFor方法

### 添加脏链表

- 1. “脏”链表是待更新的链表
- 2.更新过后就不“脏”了
- 3._active=false 的时候直接返回

### 调用 window.scheduleFrame()

- native 方法
- 按照 Framework 的模型包装、抽象然后分发
- WidgetsFlutterBinding 正是粘连 Flutter engine 和上层 Framework 的“胶水”
- UI 的绘制逻辑是在 Render 树中实现的

### 更新帧信号来临从而刷新需要重构的界面

- "scheduleBuildFor 是把一个 element 添加到 _dirtyElements 链表
- 以便当[WidgetsBinding.drawFrame]中调用 buildScope 的时候能够重构 element
- onBuildScheduled()是一个 BuildOwner 的回调"
- 在 drawFrame 中调用 buildOwner.buildScope(renderViewElement)更新 elements

# 图：

![](https://user-gold-cdn.xitu.io/2020/1/1/16f608e2ec41b049?w=2763&h=1065&f=png&s=275054)

