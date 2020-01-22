# 开发路由管理框架
看完之前两篇我们学会了：
* 路由入栈和路由出栈；
* 路由记录；
* 自定义路由动画；
* 路由传参和回退路由；
* 使用NavigatorKey进行路由管理；

那么我们今天就用之前的知识来自己开发一个属于自己的路由管理框架，这节所用到的知识就是路由封装方法，
这样使用起来只需传个新页面即可跳转了，或者随便传个自己想要的参数即可实现不一样的路由过度动画了；

# 开干
##### 创建：
```bash
flutter create --template=package nav_router
```
`nav_router`为插件名字，我这边用我自己的。

##### 路由过度动画枚举：
```dart
enum RouterType {
  material, // 默认
  cupertino, // cupertino风格
  slide, // 滑动
  scale, // 缩放
  rotation, // 旋转
  size, // 大小尺寸
  fade, // 渐变
  scaleRotate, // 缩放和旋转
}
```
##### 定义NavigatorKey：
```dart
final navGK = new GlobalKey<NavigatorState>();
```

##### 封装跳转方法：
目前演示`Push`方法，其他方法可查看本文最后的git项目地址或者查看`Navigator`自己慢慢补上；
```dart
/// 接收一个页面
/// 路由动画类型如果没传则默认为cupertino风格
Future<dynamic> routePush(Widget page,
    [RouterType type = RouterType.cupertino]) {
  Route route = routerUtil(type: type, widget: page);
  // 使用navGK进行路由跳转
  return navGK.currentState.push(route);
}
```
##### routerUtil：
用`switch`来判断应该使用哪个路由过度动画；
```dart
Route routerUtil({RouterType type, widget}) {
  Route route;
  // 使用type来case
  switch (type) {
    // 如果type为RouterType.material
    case RouterType.material:
      // 执行materialRoute方法
      route = materialRoute(widget);
      break;// 后面的以此类推
    case RouterType.cupertino:
      route = cupertinoRoute(widget);
      break;
    case RouterType.slide:
      route = slide(widget);
      break;
    case RouterType.scale:
      route = scale(widget);
      break;
    case RouterType.rotation:
      route = rotation(widget);
      break;
    case RouterType.size:
      route = size(widget);
      break;
    case RouterType.fade:
      route = fade(widget);
      break;
    case RouterType.scaleRotate:
      route = scaleRotate(widget);
      break;
  }
// 返回最终的route
  return route;
}
```
##### router：
那么我们case的需要执行的这些方法也是需要封装下；
```dart
// cupertino风格路由动画
Route cupertinoRoute(widget) {
  return new CupertinoPageRoute(
    builder: (BuildContext context) => widget,
    settings: new RouteSettings(
      name: widget.toStringShort(),
      isInitialRoute: false,
    ),
  );
}
// material风格路由动画
Route materialRoute(widget) {
  return new MaterialPageRoute(
    builder: (BuildContext context) => widget,
    settings: new RouteSettings(
      name: widget.toStringShort(),
      isInitialRoute: false,
    ),
  );
}
// 滑动路由动画
Route slide(widget) {
  return SlideRightRoute(page: widget);
}
// 缩放路由动画
Route scale(widget) {
  return ScaleRoute(page: widget);
}
// 旋转路由动画
Route rotation(widget) {
  return RotationRoute(page: widget);
}
// 尺寸大小路由动画
Route size(widget) {
  return SizeRoute(page: widget);
}
// 渐变路由动画
Route fade(widget) {
  return FadeRoute(page: widget);
}
// 缩放加旋转路由动画
Route scaleRotate(widget) {
  return ScaleRotateRoute(page: widget);
}
```
##### 动画执行类：
这里因为太多了我暂时就放下渐变和缩放的；
```dart
import 'package:flutter/material.dart';

// 缩放路由动画
class ScaleRoute extends PageRouteBuilder {
// 接收的页面page
  final Widget page;
  // 构造
  ScaleRoute({this.page})
      : super(
    pageBuilder: (
        BuildContext context,
        Animation<double> animation,
        Animation<double> secondaryAnimation,
        ) =>
    page,
    transitionsBuilder: (
        BuildContext context,
        Animation<double> animation,
        Animation<double> secondaryAnimation,
        Widget child,
        ) =>
        ScaleTransition(
          scale: Tween<double>(
            begin: 0.0, // 开始
            end: 1.0, // 结束
          ).animate(
            CurvedAnimation(
              parent: animation,
              curve: Curves.fastOutSlowIn,
            ),
          ),
          child: child, // 页面存放
        ),
  );
}

// 渐变路由动画
class FadeRoute extends PageRouteBuilder {
  // 接收的页面page
  final Widget page;
  // 构造
  FadeRoute({this.page})
      : super(
    pageBuilder: (
        BuildContext context,
        Animation<double> animation,
        Animation<double> secondaryAnimation,
        ) =>
    page,
    transitionsBuilder: (
        BuildContext context,
        Animation<double> animation,
        Animation<double> secondaryAnimation,
        Widget child,
        ) =>
        FadeTransition(
          opacity: animation, // 渐变的透明度
          child: child,// 页面存放
        ),
  );
}
```

更多路由动画直接看：
* [https://github.com/fluttercandies/nav_router/tree/master/lib/routers](https://github.com/fluttercandies/nav_router/tree/master/lib/routers)

# 使用
##### 引入
假设我是在插件的`example`目录使用，那么我就直接path方式引入；
```yaml
nav_router:
    path: ../
```
##### 导入
```dart
import 'package:nav_router/nav_router.dart';
```
##### 配置
因为我们使用的是`navigatorKey`形式跳转，所以必须在`MaterialApp`的`navigatorKey`配置好`Key`，否则无法使用；
```dart
void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'NavRoute',
      navigatorKey: navGK, // 必须配置
      home: new MyHomePage(),
    );
  }
}
```
##### 路由跳转1：
```dart
routePush(new NewPage());
```
##### 路由跳转2：
```dart
routePush(new NewPage(), RouterType.fade);
```

# End
本文路由管理框架GitHub项目地址：
* [https://github.com/fluttercandies/nav_router](https://github.com/fluttercandies/nav_router)
