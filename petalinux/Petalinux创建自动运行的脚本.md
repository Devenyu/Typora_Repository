# Petalinux创建自动运行的脚本

## ①在Petalinux中创建apps

```markdown
petalinux-create -t apps --template install -n autorunapp --enable	
```

修改项目工程目录/project-spec/meta-user/recipes-apps/autorunapp路径下的autorunapp.bb文件内容

具体内容如下所示：

```
#
# This file is the autorunapp recipe.
#

SUMMARY = "Simple autorunapp application"
SECTION = "PETALINUX/apps"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://autorunapp \
	"

S = "${WORKDIR}"

FILESEXTRAPATHS_prepend := "${THISDIR}/files:"

inherit update-rc.d

INITSCRIPT_NAME = "autorunapp"

INITSCRIPT_PARAMS = "start 99 S ."

do_install() {
        install -d ${D}${sysconfdir}/init.d
        install -m 0755 ${S}/autorunapp ${D}${sysconfdir}/init.d/autorunapp
}

FILES_${PN} += "${sysconfdir}/*"

```

修改项目工程目录/project-spec/meta-user/recipes-apps/autorunapp/files 下的autorunapp文件内容

如下所示：

```markdown
#!/bin/sh

echo "Begin Auto Run ...(DEBUG INFO) "

/media/sd-mmcblk0p1/./autorun.sh &

echo "End Auto Run ...(DEBUG INFO) "
```

## ②修改基板/etc/rc5.d/路径中的S99rmnologin.sh文件

在文件的末尾添加

```markdown
/media/sd-mmcblk0p1/./autorun.sh &
```

在/media/sd-mmcblk0p1路径中添加文件autorun.sh

```markdown
ifconfig eth0 192.168.0.100
```

