

# 分析

Flutter状态类：
* StatelessWidget：无状态类，没有状态更新，界面一经创建无法更改；
* StatefulWidget：有状态类，当状态改变，调用`setState()`方法会触发`StatefulWidget`的UI状态更新，自定义继承`StatefulWidget`的子类须重写`createState()`方法。

> 也就是只有当我们的类是有状态类的时候才能进行状态刷新，setState也是在State（有状态类）类里

# 解析 ： framework.dart文件State类

调用 `setState()` 必须是没有调用过 `dispose()` 方法，不然出错，可通过` mounted `属性来判断调用此方法是否合法。
```dart
if (mounted) {
  setState(() {});
}
```
setState方法
```dart
void setState(VoidCallback fn) {
    ...
    _element.markNeedsBuild();  
  }
```
`setState`方法除了一些条件判断就是：`_element.markNeedsBuild();`
那我们看看`markNeedsBuild`。
## Element 类 markNeedsBuild方法
```dart
  void markNeedsBuild() {
    assert(_debugLifecycleState != _ElementLifecycle.defunct);
    // 由于一帧做两次更新有点低效，所以在如果`_active=false` 的时候直接返回。
    if (!_active)
      return;//返回
     ...
    if (dirty)
      return;
     // 设置`_dirty = true `
    _dirty = true;
    //调用scheduleBuildFor方法
    owner.scheduleBuildFor(this);
  }

```
将 element 元素标记为“脏”,并把它添加到全局的“脏”链表里,以便在下一帧更新信号时更新.
* 这里的“ `脏`”链表是待更新的链表，更新过后就不“脏”了。

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
    _dirtyElements.add(element);//把element添加到脏元素链表
    element._inDirtyList = true;
    assert(() {
      if (debugPrintScheduleBuildForStacks)
        debugPrint('...dirty list is now: $_dirtyElements');
      return true;
    }());
  }
  ```
把一个 element 添加到 `_dirtyElements` 链表，主要为了方便当`WidgetsBinding.drawFrame`中调用 buildScope 
的时候能够重构 element。`onBuildScheduled()`是一个 BuildOwner 的回调。

onBuildScheduled回调在WidgetsBinding的initInstances里初始化。
```dart
mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    // 这里
    buildOwner.onBuildScheduled = _handleBuildScheduled; // 赋值onBuildScheduled
    window.onLocaleChanged = handleLocaleChanged;window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
 SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
SystemChannels.system.setMessageHandler(_handleSystemMessage);  FlutterErrorDetails.propertiesTransformers.add(transformDebugCreator);
  }
}
```
可以看到Flutter应用启动过程初始化WidgetsBinding时`buildOwner.onBuildScheduled`回调等于了
`_handleBuildScheduled`，那现在来看看这个`_handleBuildScheduled`方法：
```dart
void _handleBuildScheduled() {
    //调用ensureVisualUpdate
    ensureVisualUpdate();
  }
```
可以看到调用`ensureVisualUpdate`方法，那我们继续走下去。

# SchedulerBinding类ensureVisualUpdate方法
```dart
  void ensureVisualUpdate() {
    switch (schedulerPhase) {
      case SchedulerPhase.idle:
      case SchedulerPhase.postFrameCallbacks:
        //当schedulerPhase为SchedulerPhase.idle，
        //SchedulerPhase.postFrameCallbacks时调用scheduleFrame()
        scheduleFrame();
        return;
      case SchedulerPhase.transientCallbacks:
      case SchedulerPhase.midFrameMicrotasks:
      case SchedulerPhase.persistentCallbacks:
        return;
    }
  }
```
schedulerPhase的初始值为SchedulerPhase.idle。SchedulerPhase是一个enum枚举类型，
分别case了`SchedulerPhase` 的 5 个枚举值：

状态|含义
--|:--:
idle|没有正在处理的帧，可能正在执行的是 WidgetsBinding.scheduleTask，scheduleMicrotask，Timer，事件 handlers，或者其他回调等
transientCallbacks|SchedulerBinding.handleBeginFrame 过程， 处理动画状态更新
midFrameMicrotasks|处理 transientCallbacks 阶段触发的微任务（Microtasks）
persistentCallbacks|WidgetsBinding.drawFrame 和 SchedulerBinding.handleDrawFrame 过程，build/layout/paint 流水线工作
postFrameCallbacks|主要是清理和计划执行下一帧的工作

# 第二个case调用scheduleFrame()方法
那我们看看`scheduler/binding.dart`文件的SchedulerBinding类`scheduleFrame()`方法
```dart
  void scheduleFrame() {
  // 这个判断表示只有当APP处于可见状态才会准备调度下一帧方法
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
  这里的 `window.scheduleFrame()`方法是一个Native方法，
  由于本人并没有扎实的原生经验所以下方借鉴了袁辉辉大佬的部分讲解，
  
#  lib/ui/window.dart文件 Window类 (Native)
```dart
void scheduleFrame() native 'Window_scheduleFrame';
```
  window是Flutter引擎中跟图形相关接口打交道的核心类。
  
# ScheduleFrame(C++)  window.cc文件
```objectivec
void ScheduleFrame(Dart_NativeArguments args) {
// 看下方 RuntimeController::ScheduleFrame
  UIDartState::Current()->window()->client()->ScheduleFrame();
}
```
通过`RegisterNatives()`完成native方法的注册，“`Window_scheduleFrame`”所对应的native方法如上所示。

# RuntimeController::ScheduleFrame
所在文件：`flutter/runtime/runtime_controller.cc`
```objectivec
void RuntimeController::ScheduleFrame() {
  client_.ScheduleFrame(); // 看下面Engine::ScheduleFrame
}
```

# Engine::ScheduleFrame
所在文件：`flutter/shell/common/engine.cc`
```objectivec
void Engine::ScheduleFrame(bool regenerate_layer_tree) {
  animator_->RequestFrame(regenerate_layer_tree);
}
```
这里推荐查看袁辉辉大佬的：[Flutter渲染机制—UI线程](http://gityuan.com/2019/06/15/flutter_ui_draw/)
文中小节[2.1]介绍Engine::ScheduleFrame()经过层层调用，最终会注册Vsync回调。 等待下一次vsync信号的到来，
然后再经过层层调用最终会调用到`Window::BeginFrame()`。

# Window::BeginFrame 
所在文件：`flutter/lib/ui/window/window.cc`
```objectivec
void Window::BeginFrame(fml::TimePoint frameTime) {
  std::shared_ptr<tonic::DartState> dart_state = library_.dart_state().lock();
  if (!dart_state)
    return;
  tonic::DartState::Scope scope(dart_state);

  int64_t microseconds = (frameTime - fml::TimePoint()).ToMicroseconds();

  DartInvokeField(library_.value(), "_beginFrame",
                  {
                      Dart_NewInteger(microseconds),
                  });

  //执行MicroTask
  UIDartState::Current()->FlushMicrotasksNow();

  DartInvokeField(library_.value(), "_drawFrame", {});
}
```
`Window::BeginFrame()`过程主要工作：

* 执行_beginFrame
* 执行FlushMicrotasksNow
* 执行_drawFrame
可见，Microtask位于beginFrame和drawFrame之间，那么Microtask的耗时会影响ui绘制过程。

# handleBeginFrame
文件所在: `lib/src/scheduler/binding.dart` SchedulerBinding类
```dart
void handleBeginFrame(Duration rawTimeStamp) {
  Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
  _firstRawTimeStampInEpoch ??= rawTimeStamp;
  _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
  if (rawTimeStamp != null)
    _lastRawTimeStamp = rawTimeStamp;

  profile(() {
    _profileFrameNumber += 1;
    _profileFrameStopwatch.reset();
    _profileFrameStopwatch.start();
  });

  //此时阶段等于SchedulerPhase.idle;
  _hasScheduledFrame = false;
  try {
    Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
    _schedulerPhase = SchedulerPhase.transientCallbacks;
    //执行动画的回调方法
    final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
    _transientCallbacks = <int, _FrameCallbackEntry>{};
    callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
      if (!_removedIds.contains(id))
        _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp, callbackEntry.debugStack);
    });
    _removedIds.clear();
  } finally {
    _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
  }
}
```
该方法主要功能是遍历`_transientCallbacks`，执行相应的Animate操作，
可通过`scheduleFrameCallback()`/`cancelFrameCallbackWithId()`来完成添加和删除成员，
再来简单看看这两个方法。

# handleDrawFrame
文件所在：`lib/src/scheduler/binding.dart` SchedulerBinding类
```dart
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  Timeline.finishSync(); // 标识结束"Animate"阶段
  try {
    _schedulerPhase = SchedulerPhase.persistentCallbacks;
    //执行PERSISTENT FRAME回调
    for (FrameCallback callback in _persistentCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp);

    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    // 执行POST-FRAME回调
    final List<FrameCallback> localPostFrameCallbacks = List<FrameCallback>.from(_postFrameCallbacks);
    _postFrameCallbacks.clear();
    for (FrameCallback callback in localPostFrameCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp);
  } finally {
    _schedulerPhase = SchedulerPhase.idle;
    Timeline.finishSync(); //标识结束”Frame“阶段
    profile(() {
      _profileFrameStopwatch.stop();
      _profileFramePostEvent();
    });
    _currentFrameTimeStamp = null;
  }
}
```

该方法主要功能：

* 遍历`_persistentCallbacks`，执行相应的回调方法，可通过`addPersistentFrameCallback()`注册，一旦注册后不可移除，后续每一次frame回调都会执行；
* 遍历`_postFrameCallbacks`，执行相应的回调方法，可通过`addPostFrameCallback()`注册，`handleDrawFrame()`执行完成后会清空`_postFrameCallbacks`内容。

# UI 的绘制逻辑【附加】
UI 的绘制逻辑是在 Render 树中实现的，所以这里还来细看 RendererBinding 的逻辑。

# RendererBinding 【附加】
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
addPersistentFrameCallback 中添加 `_handlePersistentFrameCallback` 最终调用了 drawFrame 而 WidgetsBinding 重写了 RendererBinding 中的 `drawFrame()` 方法。最终发现我们又回到了 WidgetsBinding 这个类中，在 WidgetsBinding 中 drawFrame 的实现如下：
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
在上面 scheduleBuildFor 方法介绍中有提到："scheduleBuildFor 是把一个 element 添加到 _dirtyElements 链表，以便当`[WidgetsBinding.drawFrame]`中调用 buildScope 的时候能够重构 element。`onBuildScheduled()`是一个 BuildOwner 的回调"。在 drawFrame 中调用 `buildOwner.buildScope(renderViewElement)`更新 elements。
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
- 2.是否可以进行刷新：mounted

### 添加脏链表 _dirty = true

- 1.“脏”链表是待更新的链表
- 2.更新过后就不“脏”了
- 3.`_active=false` 的时候直接返回

### 管理类

- 1.告诉管理类方法自己需要被重新构建：
- owner.scheduleBuildFor(this)


### 调用 window.scheduleFrame() =》native 方法

- RegisterNatives()完成native方法的注册
- 最终会注册Vsync回调。 等待下一次vsync信号的到来，
- 然后再经过层层调用最终会调用到 Window::BeginFrame()
- UI 的绘制逻辑是在 Render 树中实现的

### 更新帧信号来临从而刷新需要重构的界面

- 在 `drawFrame` 中调用 `buildOwner.buildScope(renderViewElement)`更新 elements

