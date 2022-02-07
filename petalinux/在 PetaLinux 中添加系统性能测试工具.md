# 在 PetaLinux 中添加系统性能测试工具

- `petalinux-config -c rootfs` 增加 `packagegroup-petalinux-benchmarks`。 benchmark 软件包中包含了多项性能测试组件。具体包含内容可以在它的描述中看到

  ```markdown
  # <petalinux_install_dir>/components/yocto/source/aarch64/layers/meta-petalinux/recipes-core/packagegroups/packagegroup-petalinux-benchmarks.bb
  
  BENCHMARKS_EXTRAS = " \
     hdparm \
     iotop \
     nicstat \
     lmbench \
     iptraf \
     net-snmp \
     lsof \
     babeltrace \
     sysstat \
     dstat \
     dhrystone \
     linpack \
     whetstone \
     iperf3 \
     "
  ```

  

