# Linux 控制LED及其驱动开发

```
cd /sys/class/gpio/
echo 906 > export
echo out > gpio906/direction
echo 1 > gpio906/value		//led熄灭
```

