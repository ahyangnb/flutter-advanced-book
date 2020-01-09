# 局部刷新优化性能

Flutter状态类：
* StatelessWidget：无状态类，没有状态更新，界面一经创建无法更改；
* StatefulWidget：有状态类，当状态改变，调用`setState()`方法会触发`StatefulWidget`的UI状态更新，自定义继承`StatefulWidget`的子类须重写`createState()`方法。

案例：
> 当我们调用有状态类的setState方法时会遍历每一个子Widget的State.build刷新状态，
这将是一笔很大的性能开销，所以我们需要使用局部刷新来进行优化。

# 普通刷新方式
```dart
class TestRoute extends StatefulWidget {
  @override
  _TestRouteState createState() => _TestRouteState();
}

class _TestRouteState extends State<TestRoute> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return new FlatButton(
      onPressed: () {
        setState(() => count++);
      },
      child: new Text('$count'),
    );
  }
}
```
一个有状态类定义一个变量然后按钮的事件调用setState让这个变量进行刷新，

# 使用GlobalKey局部刷新方式
我们还是用上面的例子，只是通过GlobalKey的方式只刷新局部的Text，
```dart
class TestRoute extends StatefulWidget {
  @override
  _TestRouteState createState() => _TestRouteState();
}

class _TestRouteState extends State<TestRoute> {
  int count = 0;

  GlobalKey<TextWidgetState> textKey = GlobalKey();

  @override
  Widget build(BuildContext context) {
    return new Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: <Widget>[
        new TextWidget(textKey), //需要更新的Text
        new FlatButton(
          onPressed: () {
            count++; // 这里我们只给他值变动，状态刷新交给下面的Key事件
            textKey.currentState.onPressed(count);
          },
          child: new Text('按钮 $count'),
        ),
      ],
    );
  }
}

// 封装的文本组件Widget
class TextWidget extends StatefulWidget {
  final Key key;

  // 接收一个Key
  TextWidget(this.key);

  @override
  State<StatefulWidget> createState() => TextWidgetState();
}

class TextWidgetState extends State<TextWidget> {
  String _text = "0";

  @override
  Widget build(BuildContext context) {
    return new Text(_text);
  }

  void onPressed(int count) {
    setState(() => _text = count.toString());
  }
}
```

# 效果：

可以明显的看到按钮的count并无变动，但需要更新的文本组件更新了值，已经完美实现了局部刷新。

<img src="../img/keystate.gif" width="30%">

# 实现原理：
textKey是一个`GlobalKey`类型的Key范型为`TextWidgetState`（封装的文本&&有状态类），
所以这个Key可以通过`currentState`方法调用到类里面的`onPressed`方法，
而`onPressed`方法刚好有调用`setState`来刷新局部状态。
