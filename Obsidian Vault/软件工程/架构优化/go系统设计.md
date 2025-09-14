![[before_refactor_time.png]]![[before_refactor.png]]![[after_refactor_time.png]]![[after_refactor.png]]

通过上图的项目pprof分析和项目启动时间在重构前和重构后的对比，明显看到由于项目协程的启动数量降低后，项目启动时间大幅下降。
这里的原因是，go-routine只有三种方式可以被终止：
- 当它完成了它的工作
- 由于不可恢复的错误，它不能继续工作
- 当它告知要被终止工作
因为go-routine不会被运行时的垃圾回收，所以无论goroutine所占用的内存有多么少，它都会消耗操作系统的资源，而在go的系统设计中，重构时要考虑移除不必要的go-routine。其次是使用统一的启动组件管理业务组件的生命周期，而不是使用init函数来加载，因为go的Init函数是无状态的，不会按照特定的顺序加载。