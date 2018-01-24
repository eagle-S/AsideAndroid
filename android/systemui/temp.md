# 底部导航栏navigationBar显示状态
## 1
frameworks\base\core\res\res\values\config.xml  config_showNavigationBar

Y:\kitkat_T3\android\frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\phone\PhoneStatusBar.java  
makeStatusBarView函数中调用

Y:\kitkat_T3\android\frameworks\base\policy\src\com\android\internal\policy\impl\PhoneWindowManager.java 
最终hasNavigationBar函数

## 2
frameworks\base\core\res\res\values\dimens.xml  navigation_bar_height/navigation_bar_height_landscape

Y:\kitkat_T3\android\frameworks\base\policy\src\com\android\internal\policy\impl\PhoneWindowManager.java 
初始化mNavigationBarHeightForRotation时用到