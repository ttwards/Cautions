在docker中配置UNO R4 WiFi时
pyocd只能在启动后侦测到设备，插拔后无法找到设备
可能与docker启动过程中的udev，modprobe等有关，暂时未知
另外，"--device"参数无效
需要指定"--privilege"
