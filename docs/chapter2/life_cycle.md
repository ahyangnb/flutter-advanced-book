# 生命周期

Flutter生命周期相对于android的Activity，ta存在于`framework.dart`的State类，
![](../img/StateFul.png)

# 实测
写个有状态类并混入WidgetsBindingObserver配合监听特殊状态及其一个按钮，调用setState，
给生命周期的方法新增打印：
```dart
import 'package:flutter/material.dart';

void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: '生命周期',
      home: new LiftCycle(),
    );
  }
}

class LiftCycle extends StatefulWidget {
  @override
  _LiftCycleState createState() => _LiftCycleState();
}

class _LiftCycleState extends State<LiftCycle> with WidgetsBindingObserver {
  int count = 0;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    print('初始化 initState');
  }

  @override
  void didUpdateWidget(LiftCycle oldWidget) {
    super.didUpdateWidget(oldWidget);
    print('组件更新 didUpdateWidget');
  }

  @override
  void reassemble() {
    super.reassemble();
    print('重新安装 reassemble');
  }

  @override
  void deactivate() {
    super.deactivate();
    print('停用  deactivate');
  }

  @override
  void dispose() {
    super.dispose();
    WidgetsBinding.instance.removeObserver(this);
    print('销毁 dispose');
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    super.didChangeAppLifecycleState(state);
    print('特殊状态 state：$state');
  }

  @override
  void setState(fn) {
    super.setState(fn);
    print('状态刷新 setState');
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(title: new Text('生命周期')),
      body: new Center(
        child: new FlatButton(
          onPressed: () => setState(() => count++),
          child: new Text('$count'),
        ),
      ),
    );
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    print('依赖改变 didChangeDependencies');
  }
}
```
然后我们现在来看看打印流程，正常打开App什么都不操作，就打印了：
```
I/flutter (15867): 初始化 initState
I/flutter (15867): 依赖改变 didChangeDependencies
I/flutter (15867): 重新安装 reassemble
I/flutter (15867): 组件更新 didUpdateWidget
```

热重载打印：
```
I/flutter (16141): 重新安装 reassemble
I/flutter (16141): 组件更新 didUpdateWidget
Reloaded 0 of 468 libraries in 186ms.
```
点击按钮打印：
```
I/flutter (16141): 状态刷新 setState
// count也+1了，说明重新调用过build。
```

# 流程图

