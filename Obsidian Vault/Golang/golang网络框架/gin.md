1. gin使用注意事项：golang中使用非法时间会报error，而gin只会在有error的时候收集到ctx.Errors中，接口本身不会报错。
![[gin接收error源码.png]]![[golang非法时间报错.png]]