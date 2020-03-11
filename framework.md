# 固件编译相关
## 一、源码相关
### 1.1 源码编译命令
####      ```repo   init	-u   <URL> ```
####      ```repo  sync -c<只下载当前分支代码> -j4<多任务>```
####      ```source	build/envsetup.sh```
####      ```lunch project_name```
####      ```make -j4<多任务>```
   
### 1.2 源码导入命令
####  ``` source&lunch完之后```
####  ```make idegen && development/tools/idegen/idegen.sh```
####  ``` 对根目录生成的android.iml配置文件进行编辑优化，减少缩短导入时间```
####  ``` Android Studio选中根目录的"android.ipr"打开项目```
### 1.3 个人遇见的问题
#### Q1：无调试按钮(Attach debugger to Android process)
  `File->Project structure→Modules→右侧+号->“Android”`
#### Q2：无进程信息，无法调试
`检查SDK的配置  Settings->Android SDK`
### 1.4 模块编译
#### 1.4.1 命令
#### ``` source&lunch完之后```
#### `mmm <module_name> -j4 `
_目录下存在mk或bp文件即可编译该目录，例如只修改了frameworks/base/services下的文件，可mmm frameworks/base/services/，而无需mmm frameworks/base/_
#### 	1.4.2 Push模块
编译成功后，会看到####build completed successfully####的字样，然后会有产物路径的提示。
```
#经过make编译后的产物，都位于/out目录，该目录下主要关注下面几个目录：
Android开发工具的产物，包含SDK各种工具，比如adb，dex2oat，aapt等在/out/host。

Android系统自带的apk文件都在out/target/product/[product_name]/system/apk目录下

针对特定设备的编译产物以及平台相关C/C++代码和二进制文件在/out/target/product/[product_name]

system.img、ramdisk.img、boot.img等镜像文件在/out/target/product/[product_name]
```
以我常用的frameworks/base 举例子，
设备环境：首先`adb root ` `adb remount` `adb disable-verity`，机器重启后再`root` `remount`一下

cd 到产物路径`out/target/product/[product_name]/system/framework/`
```
adb push framework.jar /system/framework/ && 
adb push boot-framework.vdex /system/framework/ && 
adb push arm64/boot* /system/framework/arm64/ &&
adb push arm/boot*  /system/framework/arm/ 
```
```         adb shell stop && adb shell start```  重启

_确保代码与手机固件同步，两者都更新到最新就好了，不然可能开不了机_
_push产物，不确定全不全对不对，可切到目录后，用ll命令把所有的修改时间符合的文件都push进去尝试，不然也可能开不了机_
  
    
## 二、 Ninja相关
### 2.1 什么是Ninja
`类似与Make的注重速度的小型编译系统，旨在提升android的编译速度，ninja默认的入口文件是build.ninja`

### 2.2 配置Ninja
android源代码根目录下：`./prebuilts/build-tools/linux-x86/bin/ninja`
配置到环境变量上：`alias ninja='./prebuilts/build-tools/linux-x86/bin/ninja'` 即可直接使用

### 2.3 使用Ninja
`生成入口文件：`  source&lunch完之后，执行`make nothing`  
`全编`： ninja -f out/build-xxx.ninja  
`模块编译（以services.jar为例）`：执行一次 `mmm frameworks/base/services/`，out目录下会生成`build-xxx-_frameworks_base_services_Android.mk.ninja`类似的文件，执行`ninja -f out/build-xxx-_frameworks_base_services_Android.mk.ninja services`即可 ，即`ninja -f <module_ninja_file> <module_name>`。  
如果修改了mk或者bp文件需重新执行一次`mmm`命令  
`push`:与1.4.2一样



