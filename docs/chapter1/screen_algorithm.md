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
```

##### 案例1：


