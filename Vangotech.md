 1、获取PartConfig.cfg中的分区名称，起始位置，结束位置, PartFS

 2、创建空的磁盘镜像文件，这里创建一个8G的软盘

 3、查找可用的Loop Num

 4、使用 losetup将磁盘镜像文件虚拟成块设备

 5、对块设备进行分区

 6、对PartFS进行判断，如果是bin文件则直接拷贝到分区中，如果是File则先格式化分区并挂载，最后将文件拷贝到该分区

 7、删除之前虚拟的块设备，将镜像拷贝到OutputDir/linux_burn_image

 	  将outputdir中的文件打包成镜像



kernel 33792s 164863s boot.img
		program 164864s 689151s ext4/program
		data 689152s 7180799s ext4
		backup 7180800s 13672446s ext4
		others 13672447s 15245311s ext4



```
Default Firmware location: ./FIRMWARE
folder mnt/ not exist so create now
PartNum=5
7456+0 records in
7456+0 records out
7818182656 bytes (7.8 GB, 7.3 GiB) copied, 471.894 s, 16.6 MB/s
fakeMMC.img
[sudo] password for devenyu: 
Available LoopNum is 23
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Warning: The resulting partition is not properly aligned for best performance.
Model: Loopback device (loopback)
Disk /dev/loop23: 7818MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      17.3MB  84.4MB  67.1MB               kernel
 2      84.4MB  353MB   268MB                program
 3      353MB   3677MB  3324MB               data
 4      3677MB  7000MB  3324MB               backup
 5      7000MB  7806MB  805MB                others

30847+1 records in
30847+1 records out
15794031 bytes (16 MB, 15 MiB) copied, 4.62495 s, 3.4 MB/s
mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 262144 1k blocks and 65536 inodes
Filesystem UUID: 0d4ac8ba-e5ce-47fb-9f4b-d47de9e3fb3b
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 811456 4k blocks and 203200 inodes
Filesystem UUID: 361bb644-e0d3-4927-b6e1-ed6157eef318
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 811455 4k blocks and 203200 inodes
Filesystem UUID: 5e87625b-2fb7-420e-b7a2-5c9fcd4c8990
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 196608 4k blocks and 49152 inodes
Filesystem UUID: ea67b881-77da-4318-b790-4684dbd6c97e
Superblock backups stored on blocks: 
	32768, 98304, 163840

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

  adding: bootimg_AutoBoot.bin	(in=11504) (out=8490) (deflated 26%)
  adding: linux_burn_image ...........................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................	(in=7818182656) (out=23742433) (deflated 100%)
  adding: PartList.cfg	(in=140) (out=103) (deflated 26%)
  adding: spi_part.cfg	(in=69) (out=48) (deflated 30%)
  adding: u-boot_AutoBoot.bin 	(in=534896) (out=268203) (deflated 50%)
total bytes=7818729265, compressed=24019277 -> 100% savings
```

​			
