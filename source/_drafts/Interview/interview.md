Q&A
1. IM优化?
2. 网络优化？
3. iOS启动优化？
4. iOS性能优化？
   1. 卡顿检测
    ping, runloop, fps

    手淘的方案
    https://yuque.antfin.com/qitaoyang.yqt/ipu56u/we4kox?
    eleme方案
    https://yuque.antfin.com/qitaoyang.yqt/ipu56u/qio94v?
    微信方案
    https://github.com/Tencent/matrix/wiki/Matrix-for-iOS-macOS-%E5%8D%A1%E9%A1%BF%E7%9B%91%E6%8E%A7%E5%8E%9F%E7%90%86

    runloop + 堆栈快照 + 退火 ：
    
    主线程runloop用来打标(enterloop + leaveloop)，子线程每个检测周期，检测打标时间有没有超过卡顿阈值，循环队列用来记录堆栈快照。
    检测周期为子线程运行周期，也是堆栈获取周期。卡顿阈值为(enterloop - leaveloop)之间的最大时间间隔。

    卡死检测 -> 被强杀检测，排除各种正常情况以外，将最后一份快照，最为卡死的堆栈。
    正常情况包括，willterminal, app升级，系统升级，系统重启，后台oom，闪退






oc basic
1. GCD
2. 


net basic



C++ basic


