# runApp

我们平常的App入口都是：
```dart
void main() => runApp(MyApp());
```
那runApp到底做了什么，怎么来执行这个MyApp的？
```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()  //确保初始化
    ..attachRootWidget(app) //附加根小部件
    ..scheduleWarmUpFrame(); //安排热身帧
}
```
runApp方法接收一个Widget类型app值，这个值是我们需要显示的界面Widget，
然后我们看到第一个是调用了`WidgetsFlutterBinding.ensureInitialized()`，
```dart
// WidgetsFlutterBinding('flutter/lib/src/widgets/binding.dart')
class WidgetsFlutterBinding extends BindingBase 
with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding,
 SemanticsBinding, RendererBinding, WidgetsBinding {
  
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```
这WidgetsFlutterBinding是一个单例类，WidgetsFlutterBinding继承了BindingBase并且with了大量的mixin,
官方给的注释：
```
基于Widgets框架的应用程序的具体绑定。
这是将框架绑定到Flutter引擎的粘合剂。

也就是说这个类是将Widget架构和Flutter底层Engine连接的桥梁。
那么 ensureInitialized() 就是负责初始化以及返回实例的。
```

# Widget到Element到RenderObject的流程
初始化后就会继续调用attachRootWidget(app)：
```dart
// WidgetsBinding (flutter/lib/src/widgets/binding.dart)
// 取得一个小部件并将其附加到[renderViewElement]
// 该方法完成了Widget到Element到RenderObject的整个关联过程
void attachRootWidget(Widget rootWidget) {
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget,
    ).attachToRenderTree(buildOwner, renderViewElement); // 看下个方法讲解
}
```
就是将传入的Widget包装到`RenderObjectToWidgetAdapter`，它继承自`RenderObjectWidget`，
负责将`Widget`、`Element`、`RenderObject`三者关联起来，其中的`RenderObject`对应前面初始化操作中创建的`renderView`。
其中`renderView`和`_renderViewElement`为`WidgetsFlutterBinding`的成员，
可以看出每个app只存在一个`renderViewElement`和`renderView`，并且一一对应。

### attachToRenderTree
```dart
// RenderObjectToWidgetAdapter（flutter/lib/src/widgets/binding.dart）
// 此方法负责创建根Element，也就是RenderObjectToWidgetElement

/// 官方注释：
///给这个小部件充气，然后将结果[RenderObject]设置为
/// [container]的子级。
///如果`element`为null，则此函数将创建一个新元素。 除此以外，
///给定的元素将安排更新以切换到此小部件。
///可以看出Element只会创建一次，后面会进行复用
///
/// [runApp]用于引导应用程序。
RenderObjectToWidgetElement<T> attachToRenderTree
(BuildOwner owner, [ RenderObjectToWidgetElement<T> element ]) {
    // 如果`element`为null
    if (element == null) {
      // 创建根Element，RenderObjectToWidgetElement
      owner.lockState(() {
        element = createElement();
        assert(element != null);
        element.assignOwner(owner);
      });
      owner.buildScope(element, () {
        // 这里会根据WidgetTree构建ElementTree
        element.mount(null, null); // 看下面的mount方法讲解
      });
    } else { // 则此函数将创建一个新元素
      element._newWidget = this;
      element.markNeedsBuild(); // markNeedsBuild在setState更新原理和流程有讲到
    }
    return element;
}
```

### mount
```dart
// RenderObjectToWidgetAdapter（flutter/lib/src/widgets/binding.dart）
@override
  void mount(Element parent, dynamic newSlot) {
    assert(parent == null); // 断言接收的parent等于空
    super.mount(parent, newSlot); 
    _rebuild();
}
```

### _rebuild
```dart
// RenderObjectToWidgetAdapter（flutter/lib/src/widgets/binding.dart）
void _rebuild() {
    try {
      // 实际上是调用updateChild更新ElementTree
      _child = updateChild(_child, widget.child, _rootChildSlot); // 查看【updateChild】
      assert(_child != null);
    } catch (exception, stack) {
      // 红屏产生的地方
      final FlutterErrorDetails details = FlutterErrorDetails(
        exception: exception,
        stack: stack,
        library: 'widgets library',
        context: ErrorDescription('attaching to the render tree'),
      );
      // 这里打印了错误栈
      FlutterError.reportError(details);
      // 这里就是创建了红屏的Widget，显示在屏幕上
      // 自定义ErrorWidget看下一个方法
      final Widget error = ErrorWidget.builder(details);
      _child = updateChild(null, error, _rootChildSlot);// 查看【updateChild】
    }
}
```

### 自定义ErrorWidget报错页面
```dart
void main() async {
  runApp(MyApp());

  /// 自定义报错页面
  ErrorWidget.builder = (FlutterErrorDetails flutterErrorDetails) {
    debugPrint(flutterErrorDetails.toString());
    return new Center(child: new Text("App错误，快去反馈给作者!"));
  };
}
```

### updateChild() 更新ElementTree
实际上该方法只执行了updateChild()，该方法至关重要，ElementTree的生成主要就在方法中实现，
我们来细看一下代码，注意代码中添加的注释：
```dart
// child表示要更新的Element，newWidget表示对应Element的Widget，
// newSlot用来标识Element的所在位置，返回该位置对应的新Element
@protected
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
  assert(() {
    // Debug下保证一个GlobalKey只对应一个Widget
    if (newWidget != null && newWidget.key is GlobalKey) {
      final GlobalKey key = newWidget.key;
      key._debugReserveFor(this);
    }
    return true;
  }());
  if (newWidget == null) {
    // 如果newWidget为空，child非空表示需要移除旧Element
    if (child != null) deactivateChild(child);
    // 将此Element的位置设为null
    return null;
  }
  if (child != null) {
    // 都不为空且是相同Widget，更新位置标识即可
    if (child.widget == newWidget) {
      if (child.slot != newSlot) updateSlotForChild(child, newSlot);
      // 更新后返回原Element
      return child;
    }
    // 若不是相同Widget则判断是否有相同的类型和相同的Key，是的话则更新Widget信息到Element
    if (Widget.canUpdate(child.widget, newWidget)) {
      if (child.slot != newSlot) updateSlotForChild(child, newSlot);
      child.update(newWidget);
      assert(child.widget == newWidget);
      assert(() {
        child.owner._debugElementWasRebuilt(child);
        return true;
      }());
      // 更新后返回原Element
      return child;
    }
    // 若不符合更新的要求，则抛弃掉原Element，抛弃掉的Element会被回收到`_inactiveElements`列表中，不会立即被销毁
    deactivateChild(child);
    assert(child._parent == null);
  }
  // 其他情况下需要创建新的Element
  return inflateWidget(newWidget, newSlot); //查看【inflateWidget】
}
```
总结

|                     | **newWidget等于空**  | **newWidget不等于空**   |
   | :-----------------: | :--------------------- | :---------------------- |
   |  **child等于空**  |  返回null.         |  返回新 [Element]. |
   |  **child不等于空**  |  旧child被删除，返回空. | 可能会更新旧的子级，返回子级或新的[Element]. |
   

### inflateWidget
```dart
///为给定的小部件创建一个元素，并将其添加为该元素的子元素给定插槽中的元素。
///
///此方法通常由[updateChild]调用，但可以调用直接由需要更精细地控制创建的子类元素。
///
///如果给定的小部件具有全局键并且已经存在一个元素有一个带有该全局键的小部件，此函数将重用该元素
///（可能从树中的其他位置移植或重新激活从无效元素列表中获取），而不是创建一个新元素。
///
///此函数返回的元素将已经被挂载并将处于“活动”生命周期状态。
@protected
Element inflateWidget(Widget newWidget, dynamic newSlot) {
  assert(newWidget != null);
  final Key key = newWidget.key;
  if (key is GlobalKey) {
    // 先使用key去被回收的列表中看看是否有可以复用的Element
    final Element newChild = _retakeInactiveElement(key, newWidget);
    if (newChild != null) {
      assert(newChild._parent == null);
      assert(() {
        _debugCheckForCycles(newChild);
        return true;
      }());
      newChild._activateWithParent(this, newSlot);
      // 找到后就复用被回收的Element，并且更新它的Child
      final Element updatedChild = updateChild(newChild, newWidget, newSlot);
      assert(newChild == updatedChild);
      return updatedChild;
    }
  }
  // 没有可以复用的Element了，创建新的
  final Element newChild = newWidget.createElement();
  assert(() {
    _debugCheckForCycles(newChild);
    return true;
  }());
  // mount安装新的Element
  newChild.mount(this, newSlot);
  assert(newChild._debugLifecycleState == _ElementLifecycle.active);
  // 返回新的child
  return newChild;
}
```
新创建的`Element`继续调用`mount`，于是又会触发新一轮的`updateChild`，
最终对应`WidgetTree`的整个`ElementTree`就构建完成了。

# 渲染回调等部分
渲染主要是在`WidgetsFlutterBinding`类开始执行的，runApp方法最后也是执行了WidgetsFlutterBinding类的
scheduleWarmUpFrame方法进行第一次绘制。
```dart
///锁定事件调度，直到调度的框架完成为止。
///也就是这次完成之后才会开始绘制其他scheduledFrame。
///如果已经使用[scheduleFrame]安排了帧，或者[scheduleForcedFrame]，此调用可能会延迟该帧。
///如果任何预定的帧已经开始或其他[scheduleWarmUpFrame]已被调用，此调用将被忽略。
///首选[scheduleFrame]在正常操作下更新显示。
void scheduleWarmUpFrame() {
    if (_warmUpFrame || schedulerPhase != SchedulerPhase.idle)
      return;

    _warmUpFrame = true;
    Timeline.startSync('Warm-up frame');
    final bool hadScheduledFrame = _hasScheduledFrame;
    // 我们在这里使用计时器来确保微任务在两者之间刷新。
    Timer.run(() {
      assert(_warmUpFrame);
      handleBeginFrame(null); // 【主要方法1】
    });
    Timer.run(() {
      assert(_warmUpFrame);
      handleDrawFrame(); // 【主要方法2】
      //我们在此帧之后调用resetEpoch
      resetEpoch();
      _warmUpFrame = false;
      if (hadScheduledFrame)
        scheduleFrame();
    });
    //锁定事件，以便触摸事件等直到预定的帧已完成。
    lockEvents(() async {
      await endOfFrame;
      Timeline.finishSync();
    });
  }
```

其实绘制主要是用到了`handleBeginFrame()`和`handleDrawFrame()`两个方法，
因为这两个方法调用由`scheduleFrameCallback`命令注册的所有需要的回调。

所以建议看这两个方法之前了解下`Frame`和`FrameCallbacks`：

##### Frame
`Frame`即每一帧的绘制过程，`engine`通过`VSync`信号不断地触发`Frame`的绘制，
实际上就是调用`SchedulerBinding`类中的`_handleBeginFrame()`和`_handleDrawFrame()`这两个方法，
这个过程中会完成动画、布局、绘制等工作。

##### FrameCallbacks
`Frame`绘制期间，有三个`callbacks`列表会被调用，这三个列表是`SchedulerBinding`类中的成员，它们的调用顺序如下：

|  顺序   | 内容  |
|  ----  | ----  |
| transientCallbacks  | 由Ticker触发和停止，一般用于动画的回调 |
| persistentCallbacks  | 永久callback，添加后无法移除，由`WidgetsBinding.instance.addPersitentFrameCallback()`注册，这个回调处理了布局与绘制工作 |
| postFrameCallbacks  | 只调一次，调用后会被系统移除，可由`WidgetsBinding.instance.addPostFrameCallback()`注册，该回调一般用于State的更新 |

### handleBeginFrame方法
代码已忽略断言部分。
```dart
// SchedulerBinding(flutter/lib/src/scheduler/binding.dart)
/// 如果给定的时间戳为null，则最后一帧的时间戳为重用。
void handleBeginFrame(Duration rawTimeStamp) {
  Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
  _firstRawTimeStampInEpoch ??= rawTimeStamp;
  _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
  if (rawTimeStamp != null) _lastRawTimeStamp = rawTimeStamp;

  _hasScheduledFrame = false;
  try {
    // transientCallbacks回调
    Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
    _schedulerPhase = SchedulerPhase.transientCallbacks;
    final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
    _transientCallbacks = <int, _FrameCallbackEntry>{};
    callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
      if (!_removedIds.contains(id))
        _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp,
            callbackEntry.debugStack);
    });
    _removedIds.clear();
  } finally {
    _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
  }
}
```
这里主要执行了`transientCallbacks`回调。

### handleDrawFrame
执行了`persistentCallbacks`和`postFrameCallbacks`回调，主要的操作都在这里。
```dart
/// handleBeginFrame之后立即调用此方法。 触发全部addPersistentFrameCallback注册的回调，通常
///驱动渲染管道，然后调用addPostFrameCallback。
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  Timeline.finishSync(); // 结束“动画”阶段
  try {
    // persistentCallbacks回调
    _schedulerPhase = SchedulerPhase.persistentCallbacks;
    for (FrameCallback callback in _persistentCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp);

    // postFrameCallbacks回调
    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    final List<FrameCallback> localPostFrameCallbacks =
        List<FrameCallback>.from(_postFrameCallbacks);
    _postFrameCallbacks.clear();
    for (FrameCallback callback in localPostFrameCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp);
  } finally {
    _schedulerPhase = SchedulerPhase.idle;
    Timeline.finishSync(); //结束帧
    assert(() {
      if (debugPrintEndFrameBanner) debugPrint('▀' * _debugBanner.length);
      _debugBanner = null;
      return true;
    }());
    _currentFrameTimeStamp = null;
  }
}
```

# 渲染操作
系统只在`persistentCallbacks`注册了一个回调，
实际上是执行`RenderBinding`类中的`drawFrame()`方法以及其子类`WidgetsBinding`类中的`drawFrame()`方法：
```dart
@protected
void drawFrame() {
    pipelineOwner.flushLayout(); // 看 【3.2.1】
    pipelineOwner.flushCompositingBits(); // 看 【3.2.2】
    pipelineOwner.flushPaint(); // 看 【3.2.3】
    renderView.compositeFrame(); // 看 【3.2.4】
    pipelineOwner.flushSemantics(); // 看 【3.2.5】
}
```
** WidgetsBinding ** 
```dart
/// 代码忽略了断言和判断
@override
void drawFrame() {
  try {
    if (renderViewElement != null)
      buildOwner.buildScope(renderViewElement);
    super.drawFrame();
    buildOwner.finalizeTree();
  } finally {
    ...
  }
  _needToReportFirstFrame = false;
}
```
`buildOwner.buildScope()`在我们的"setState更新原理和流程"有讲到过，可以直接搜索。
> 该方法会将被标记为dirty的Element进行重新构建。

回收被抛弃的Element的列表_inactiveElements最后会调用buildOwner.finalizeTree()彻底清除掉。

### 3.2.1 pipelineOwner.flushLayout()
该方法更新所有脏渲染对象的布局等信息。
```dart
/// 布局信息在绘制之前已清理，因此渲染对象将出现在屏幕上的最新位置。
/// 
/// 当RenderObject的宽高等布局相关的属性被set时（通过更改Widget的属性），
/// 它会被添加到_nodesNeedingLayout列表中，以标记为需要重新进行layout。
/// 这里遍历了该列表，并调用_layoutWithoutResize()进行布局
void flushLayout() {
  ...
  try {
    while (_nodesNeedingLayout.isNotEmpty) {
      final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
      _nodesNeedingLayout = <RenderObject>[];
      for (RenderObject node in dirtyNodes
        ..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
        if (node._needsLayout && node.owner == this)
          node._layoutWithoutResize();
      }
    }
  } finally {
    ...
  }
}
```

### 3.2.2 flushCompositingBits
在flushLayout之后和之前作为渲染管道的一部分调用
```dart
/// 用于判断RenderObject是否拥有自己的layer，如果该状态变化了，就会将该RenderObject标记为需要进行重绘的，
/// 然后在下面flushPaint()方法中进行重绘。
void flushCompositingBits() {
  ...
  _nodesNeedingCompositingBitsUpdate
      .sort((RenderObject a, RenderObject b) => a.depth - b.depth);
  for (RenderObject node in _nodesNeedingCompositingBitsUpdate) {
    if (node._needsCompositingBitsUpdate && node.owner == this)
      node._updateCompositingBits();
  }
  _nodesNeedingCompositingBitsUpdate.clear();
  ...
}
```

### 3.2.3 flushPaint
```dart
/// 该方法就是进行绘制的地方，可以看出它不是重绘了所有RenderObject，而是只重绘了被标记为dirty的RenderObject，
/// 这些RenderObject会调用engine下的skia库进行绘制。
void flushPaint() {
  final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
  _nodesNeedingPaint = <RenderObject>[];
  // 以相反的顺序对脏节点进行排序（最深的优先）。
  for (RenderObject node in dirtyNodes
    ..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
    assert(node._layer != null);
    if (node._needsPaint && node.owner == this) {
      if (node._layer.attached) {
        PaintingContext.repaintCompositedChild(node);
      } else {
        node._skippedPaintingOnLayer();
      }
    }
  }
  ...
}
```
### 3.2.4 compositeFrame
```dart
// RenderView(flutter/lib/src/rendering/view.dart)

///将合成的层树上载到引擎。
///实际上使渲染管道的输出出现在屏幕上。
void compositeFrame() {
  Timeline.startSync('Compositing', arguments: timelineWhitelistArguments);
  try {
    final ui.SceneBuilder builder = ui.SceneBuilder();
    final ui.Scene scene = layer.buildScene(builder);
    if (automaticSystemUiAdjustment) _updateSystemChrome();
    _window.render(scene);
    scene.dispose();
  } finally {
    Timeline.finishSync();
  }
  ...
}
```
将画好的layer传给engine，该方法调用结束之后，手机屏幕就会显示出内容了。

### 3.2.5 flushSemantics
```dart
/// Semantics用于将一些Widget的信息传给系统用于搜索、App内容分析等场景，这与Flutter绘制流程关系不大。
void flushSemantics() {
  final List<RenderObject> nodesToProcess = _nodesNeedingSemantics.toList()
    ..sort((RenderObject a, RenderObject b) => a.depth - b.depth);
  _nodesNeedingSemantics.clear();
  for (RenderObject node in nodesToProcess) {
    if (node._needsSemanticsUpdate && node.owner == this)
      node._updateSemantics();
  }
  _semanticsOwner.sendSemanticsUpdate();
}
```

# 总结
其实就是根据传入的Widget生成对应的`ElementTree`和`RenderTree`，之后开始进行首帧的布局和绘制。
其中`Widget`用来描述页面的属性，这个对象是十分轻量级的且是不可变的，同一个`Widget`可以描述多个`Element`的配置，
`Element`同时持有了`Widget和RenderObject`，它汇总了所有的属性信息，重绘时只将需要修改的部分通知到`RenderObject`。
对于普通开发者，只需要关注最上层的`Widget`就可以了，十分简单高效。

> 由于本人能力有限，部分讲解借鉴了`超丶赛亚叼 `的，在此致敬。

