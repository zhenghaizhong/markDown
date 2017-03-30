# USE

### 1.cd android-sdk/platform-tools/systrace
###  2.python systrace.py --time=5 -o mynewtrace.trace sched gfx view wm am
注：--time=？为抓取时间，不添加便自己控制。wm为Windowsmanager ，am为Activity Manager，一般应用的分析需添加am

### 3.使用chrome打开后，w放大，s缩小 ，m获取单个方法或配合ctrl+左键来获取某个时间段的情况，左键单击能显示该方法的信息