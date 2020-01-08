# 屏幕适配之屏幕算法

既然是算法适配就必然少不了获取屏幕宽高，我们用的就是媒体查询（MediaQuery），
下面是封装方法过后的，当然直接使用也是可以的：
```dart
// 整屏宽度
double winWidth(BuildContext context) {
  return MediaQuery.of(context).size.width;
}
// 整屏高度
double winHeight(BuildContext context) {
  return MediaQuery.of(context).size.height;
}
// 顶部padding
double winTop(BuildContext context) {
  return MediaQuery.of(context).padding.top;
}
// 底部padding
double winBottom(BuildContext context) {
  return MediaQuery.of(context).padding.bottom;
}
// 左边padding
double winLeft(BuildContext context) {
  return MediaQuery.of(context).padding.left;
}
// 右边padding
double winRight(BuildContext context) {
  return MediaQuery.of(context).padding.right;
}
// 状态栏高度
double statusBarHeight(BuildContext context) {
  return MediaQueryData.fromWindow(window).padding.top;
}
// 工具栏高度
double navigationBarHeight(BuildContext context) {
  return kToolbarHeight;
}
// AppBar高度
double topBarHeight(BuildContext context) {
  return kToolbarHeight + MediaQueryData.fromWindow(window).padding.top;
}
// 键盘高度：键盘未弹出则为0
double keyBordHeight(BuildContext context) {
  return MediaQuery.of(context).viewInsets.bottom;
}
```
