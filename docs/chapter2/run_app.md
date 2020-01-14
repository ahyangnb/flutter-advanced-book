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


更新中。。。