# Architecture Components & MVVM 摘要学习    
            部分非原创，记录自己的学习
## 一.MVC、MVP、MVVM
* **MVC**：Activity/Fragment责任不明，同时兼顾了View与Controller，导致代码过于臃肿 、V与C一一对应，较难复用、增加系统结构和实现的复杂性

* **MVP**:View责任明确、Model层不再持有View、引入大量的接口、V和P相互持有

* **MVVM**:Google官方推荐的架构模式- -..   、本质是数据驱动

## 二.Architecture Components
* **Lifecycle**：管理组件生命周期（解耦、可移植）; 用于精简V层生命周期，个人项目中未使用到- -。。
* **LiveData**：具有生命周期感知且可被观察订阅的数据保持类；如：LiveData<List<String>> 这样子使用，实现数据驱动，基本在ViewModel中创建
* **ViewModel**：	