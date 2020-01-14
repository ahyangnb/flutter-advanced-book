# 根据控件位置弹出对话框

##### 实现效果

<img src="../img/offset_widget.gif" width="30%">

首先我们要知道如何获取控件尺寸和位置信息，

* 插件必须渲染好,

```dart
final RenderBox box = globalKey.currentContext.findRenderObject();
final size = box.size;  // 获取尺寸
final topLeftPosition = box.localToGlobal(Offset.zero);
return topLeftPosition.dy;
```

* 可以通过 `context.size`获取当前控件的尺寸和位置offset信息

下面是示例，通过`context.size.height`可以拿到child控件的高度

```dart
class HeightReporter extends StatelessWidget {
  final Widget child;
 
  HeightReporter({this.child});
 
  @override
  Widget build(BuildContext context) {
  // 手势识别组件包裹一个Widget孩子
    return new GestureDetector(
      child: child,  // 获取尺寸高度并打印
      onTap: () =>  print('Height is ${context.size.height}'),
    );
  }
}
```

正在更新中。。。。