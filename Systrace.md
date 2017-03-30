# USE

### 1.cd android-sdk/platform-tools/systrace
###  2.python systrace.py --time=5 -o mynewtrace.trace sched gfx view wm am
注：--time=？为抓取时间，不添加便自己控制。wm为Windowsmanager ，am为Activity Manager，一般应用的分析需添加am
