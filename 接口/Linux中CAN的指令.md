# Linux中CAN的指令

配置CAN的波特率

```markdown
ip link set can0 type can bitrate 500000
```

如果can bus-off设置自动重启的时间

```markdown
ip link set can0 type can restart-ms 100
```

启动CAN设备

```markdown
ip link set can0 up
```

关闭CAN设备

```markdown
ip link set can0 down
```

查看 can0 的配置

```markdown
ip -details link show can0
```

查看 can0 的比特率配置等,以及统计数据(接收/发送/出错帧等)

```markdown
ip -details -statistics link show can0 
```

发送报文

```markdown
cansend can0 123#2233
```

接收报文

```markdown
candump can0 
```

开机自动配置

/etc/network/interface

```markdown
		auto can0
		iface can0 inet manual
		#pre-up ip link set $IFACE type can bitrate 125000 listen-only off
		pre-up /ip link set $IFACE type can bitrate 125000 triple-sampling on
		up /sbin/ifconfig $IFACE up
		down /sbin/ifconfig $IFACE down


```

