# 给petalinux打补丁的方法

## 如何管理Petalinux的源代码

首先要了解的是进入Petalinux的源代码是如何管理的。每个Petalinux版本都是由几个Git仓库的特定提交组成。在这篇文章中，我们将只讨论其中的四个仓库；它们分别是FSBL、U-Boot、Device Tree和内核的源代码。

- Standalone（including FSBL）：embeddedsw
- U-Boot：u-boot-xlnx
- Device tree：device-tree-xlnx
- Kernel：linux-xlnx

## 给Petalinux打补丁的一般程序

1、使用git clone 克隆源代码，并签出一个特定发布的标签。

2、修改我们本地的源代码副本。

3、使用git diff生成一个补丁文件。

4、在我们的Petalinux项目配置（project-spec）包含该补丁文件。

## 为FSBL打补丁

```markdown
git clone https://github.com/Xilinx/embeddedsw.git
cd embeddedsw/
git checkout tags/xilinx-v2020.2
git branch fsbl_mods_2020.2
git checkout fsbl_mods_2020.2
```

- 在以下路径中对FSBL进行修改：lib/sw_apps/zynq_fsbl/src/

- 将修改的文件提交。

```markdown
git add lib/sw_apps/zynq_fsbl/src/main.c
git commit -m "Patch to Config phy chip" -s
```

- 创建补丁

```markdown
git format-patch -1
```

- 当补丁创建完成之后我们需要将补丁文件拷贝到Petalinux中的以下路径：`/project-spec/meta-user/recipes-bsp/fsbl/files/`.

并且在/project-spec/meta-user/recipes-bsp/fsbl/中添加fsbl_%.bbappend文件，具体的代码如下：

```markdown
# Patch for FSBL
# Note: do_configure_prepend task section is required only for 2017.1 release
# Refer https://github.com/Xilinx/meta-xilinx-tools/blob/rel-v2017.2/classes/xsctbase.bbclass#L29-L35
  
do_configure_prepend() {
    if [ -d "${S}/patches" ]; then
       rm -rf ${S}/patches
    fi
  
    if [ -d "${S}/.pc" ]; then
       rm -rf ${S}/.pc
    fi
}
  
SRC_URI_append = " \
	file://0001-Patch-to-Reset-the-phy-chip.patch \
        "
  
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
  

# Note: This is not required if you are using Yocto
# CAUTION!: EXTERNALXSCTSRC and EXTERNALXSCTSRC_BUILD is required only for 2018.2 and below petalinux releases
EXTERNALXSCTSRC = ""
EXTERNALXSCTSRC_BUILD = ""
```

## 为U-Boot打补丁

```markdown
git clone https://github.com/Xilinx/u-boot-xlnx.git
cd u-boot-xlnx
git checkout xilinx-v2020.2
```

- 对版本库拷贝的本地进行修改，并保存修改。

- 通过在本地repo目录下运行此命令生成补丁文件：git diff > uboot.patch

- 将补丁文件复制到/project-spec/meta-user/recipes-bsp/u-boot/files/

- 编辑文件 /project-spec/meta-user/recipes-bsp/u-boot/u-boot-xlnx_%.bbappend 并添加以下一行。SRC_URI += "file://uboot.patch"，具体的代码如下：

  ```markdown
  FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
   
  SRC_URI += "file://platform-top.h"
  SRC_URI += "file://uboot.patch"
   
  do_configure_append () {
  	if [ "${U_BOOT_AUTO_CONFIG}" = "1" ]; then
  		install ${WORKDIR}/platform-auto.h ${S}/include/configs/
  		install ${WORKDIR}/platform-top.h ${S}/include/configs/
  	fi
  }
   
  do_configure_append_microblaze () {
  	if [ "${U_BOOT_AUTO_CONFIG}" = "1" ]; then
  		install -d ${B}/source/board/xilinx/microblaze-generic/
  		install ${WORKDIR}/config.mk ${B}/source/board/xilinx/microblaze-generic/
  	fi
  }
  ```

  ## 为设备树打补丁

  ```markdown
  git clone https://github.com/Xilinx/device-tree-xlnx.git
  cd device-tree-xlnx
  git checkout xilinx-v2020.2
  ```
  
- 对本地副本进行修改。

- 通过在本地版本的目录下运行此命令来创建补丁文件：git diff > devtree.patch。使用一个描述性的文件名，但保留 .patch 的扩展名。

- 将补丁文件复制到/project-spec/meta-user/recipes-bsp/device-tree/files/

- 在文件编辑器中打开这个文件。/project-spec/meta-user/recipes-bsp/device-tree/device-tree.bbappend 并在文件中添加这一行。SRC_URI += "file://devtree.patch"，具体的代码如下：

```markdown
FILESEXTRAPATHS_prepend := "${THISDIR}/files:"
 
SRC_URI += "file://my-dev-tree.dtsi"
SRC_URI += " file://emacps.patch"
 
python () {
    if d.getVar("CONFIG_DISABLE"):
        d.setVarFlag("do_configure", "noexec", "1")
}
 
export PETALINUX
do_configure_append () {
	script="${PETALINUX}/etc/hsm/scripts/petalinux_hsm_bridge.tcl"
	data=${PETALINUX}/etc/hsm/data/
	eval xsct -sdx -nodisp ${script} -c ${WORKDIR}/config \
	-hdf ${DT_FILES_PATH}/hardware_description.${HDF_EXT} -repo ${S} \
	-data ${data} -sw ${DT_FILES_PATH} -o ${DT_FILES_PATH} -a "soc_mapping"
}
```

## 为内核打补丁

```markdown
git clone https://github.com/Xilinx/linux-xlnx.git
cd linux-xlnx
git checkout xlnx_rebase_v5.4_2020.2
```

- 修改本地repo中的代码，并保存修改内容。
- 在本地版本库的目录下运行此命令来创建补丁文件：git diff > kernel.patch。为补丁文件使用一个描述性的名字，但保留.patch扩展名。
- 将补丁文件复制到 /project-spec/meta-user/recipes-kernel/linux/linux-xlnx/ 中。
- 编辑这个文件。/project-spec/meta-user/recipes-kernel/linux/linux-xlnx_%.bbappend 并添加以下一行。SRC_URI += "file://kernel.patch",具体的代码如下：

```markdown
FILESEXTRAPATHS_prepend := "${THISDIR}/${PN}:"
 
SRC_URI += "file://user.cfg \
            file://kernel.patch \
            "
```

