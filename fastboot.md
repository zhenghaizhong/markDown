# fastboot

## 更新升级
`< waiting for any device >`
sudo chown root:root c/bin/fastboot
sudo chmod +s /usr/bin/fastboot
### Android Q 之前：

        重启手机并进入刷机状态 adb reboot bootloader
        选择要刷入的分区img:
		fastboot flash boot boot.img
		fastboot flash recovery recovery.img
		fastboot flash ramdisk ramdisk.img
		fastboot flash userdata userdata.img
		fastboot flash system system.img
        重启手机 fastboot reboot 
        
### Andoird Q:
`system.img`和` vendor.img` 进行合并变成了` super.img` 动态分区  ( system/vendor 大小可变,但不超过 super 总大小,故称动态分区)

_需更新fastboot工具，Android Q SDK sdk/platform-tools下的即可_

#### 支持 DP 系统后，如何单独烧写 system.img 和 vendor.img?
`adb  reboot fastboot,进入 fastbootd 模式`  
 `  fastboot flash system system.img`  
 ` fastboot flash vendor vendor.img`
                     
#### fastboot (recovery)模式和 fastbootd 模式如何相互切换？
 `进 fastbootd 模式的命令是 fastboot reboot fastboot`  
 `进 recovery 模式的命令是 fastboot oem reboot recovery`
 
#### 支持 DP 系统后， 如何更新 super 分区？
`adb reboot bootloader`  
`fastboot flash super super.img`  
`sudo fastboot --disable-verity flash vbmeta vbmeta.img`  
`sudo fastboot --disable-verity flash vbmeta_system vbmeta_system.img`  
`fastboot reboot`
