---
title: TA 开发问题
date: 2020-02-19 17:09:12
updated: 2020-02-19 17:09:12
tags:
 - trustZone
 - 安全
categories: 安全
keywords: trustZone
copyright: true
---

## (待解决)1.  APP 访问 /dev/tee0权限问题

详细描述： android app 权限低于10000, /dev/tee0权限：

`crw------- root     root     248,   0 1912-12-20 04:34 tee0`

暂时解决办法： chmod 666 /dev/tee0

弊端：所有app都能读写/dev/tee0

思路：app添加`android:sharedUserId="android.uid.system"` 使app为系统app，签名时android studio平台签名出现问题，可在源码中编译

<!-- more -->

## 2. TA参数只能传4个

operation params 结构体默认为4个

解决办法：1. 修改optee源码，但不具有通用性

2. 将几个参数包装成结构体，注意结构体内不能有指针成员；或同类型合并同时传入长度

## 3. 运行时出现trans fault 

知识相关：optee运行时地址布局，页表映射机制

解决办法： TA_STACK_SIZE 指定了栈大小，而运行时超过该大小，将其值改大。

待解决：char *serial ="/proc/cpuinfo"; TEE_MemMov(cpu, serial, 13)出现错误。



## 4. 全局变量

详细描述： 每次进入SW重新加载TA，之前运行时获得的全局变量已被销毁。

解决方法：全局变量写入文件保存，用时读取，但效率太低。

寻找更好方法中.................. 

**user_ta_header_defines.h** 中：

```c
#define TA_FLAGS                    (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION | TA_FLAG_EXEC_DDR | TA_FLAG_INSTANCE_KEEP_ALIVE)
```

## 5. 参数传递

字符串传递时末尾`‘\0’`  不会传递，不能使用`strlen()` 计算长度，将长度同时传递。

`strlen` 长度不包括末尾`\0` 



---

## QSEE TA

### 1. 添加新TA [helloCA]

路径core/bsp/trustzone/qsapps/ 下新建目录结构：

![file](file.png)

SConscript内容相应如下：

``` python
import os
Import('env')

env = env.Clone()
#env.Append(CFLAGS = " -Werror ")

#------------------------------------------------------------------------------
# Check if we need to load this script or just bail-out
#------------------------------------------------------------------------------
# alias - First alias is always the target then the other possible aliases
aliases = [
   'helloCA', 'all'
]

env.InitImageVars(
  alias_list = aliases,       # list of aliases, unique name index [0]
  proc = 'scorpion',               # proc settings
  config = 'apps',            # config settings
  build_tags = ['APPS_PROC',
                'HELLOCA_IMAGE'
               ],
)

if not env.CheckAlias():
   Return()
#------------------------------------------------------------------------------
# Configure and load in USES and path variables
#------------------------------------------------------------------------------
env.LoadToolScript('${BUILD_ROOT}/core/bsp/build/scripts/secure_app_builder.py')
env.InitBuildConfig()
env.Append(OUT_DIR= os.getcwd())
sconspath = env.subst('${BUILD_ROOT}/core/securemsm/trustzone/qsapps/helloCA/src/SConscript')
env.Replace(SRC_SCONS_ROOT = sconspath.split('SConscript')[0])
SConscript( sconspath, exports='env',)
env.Deploy('SConscript')
```

   路径core/securemsm/trustzone/qsapps/为项目源码文件夹，新建文件结构：

![file1](file1.png)

SConscript内容如下：

``` python
Import('env')

if env.has_key('USES_NO_CP'):
  env.Append(CCFLAGS = ' -DUSES_NO_CP ')
  
#-------------------------------------------------------------------------------
# Compiler, object, and linker definitions
#-------------------------------------------------------------------------------
includes = [
  "${BUILD_ROOT}/core/api/kernel/libstd/stringl",
  "${BUILD_ROOT}/core/securemsm/trustzone/qsapps/helloCA/inc",
  '${BUILD_ROOT}/core/securemsm/sse/log/inc',
  '${BUILD_ROOT}/coreapi/securemsm/trustzone/qsee'
]

#------------------------------------------------------------------------------
# We need to specify "neon" to generate SIMD instructions in 32-bit mode
#------------------------------------------------------------------------------
if env['PROC'] == 'scorpion':
  env.Append(CCFLAGS = " -mfpu=neon ")


target_name = 'helloCA'
app_name = 'helloCA'



# enable logging
env.Append(CPPDEFINES = ['-DSSE_LOG_GLOBAL'])
env.Append(CPPDEFINES = ['-DSSE_LOG_FILE'])
env.Append(CPPDEFINES = ['-DLOG_TAG=helloCA'])

#----------------------------------------------------------------------------
# App core Objects
#----------------------------------------------------------------------------
sources = [
  'app_main.c',
]
#-------------------------------------------------------------------------------
# Add metadata to image
#-------------------------------------------------------------------------------
md = {
  'appName':    app_name,
  'privileges': ['default'],
  'acceptBufSize': 8192,
}

if env['PROC'] == 'scorpion':
  md['memoryType'] = 'Unprotected'

deploy_header_files = env.Glob('../inc/*')

helloCA_units = env.SecureAppBuilder(
  sources = sources,
  includes = includes,
  metadata = md,
  image = target_name,
  deploy_sources = sources + ['SConscript'] + deploy_header_files
)

for image in env['IMAGE_ALIASES']:
  op = env.Alias(image, helloCA_units)
```

  配置文件build/ms路径下build_config.xml中 \<target name="common"\> 标签下添加

``` xml
<file name="helloCA"
      recompile="true" >
    <artifact name="helloCA" />
    <mapreport path="core/bsp/trustzone/qsapps" />
    <param name="USES_FLAGS" value="USES_NO_CP" />
</file>
```

相同路径下新建编译脚本helloCA.sh:

``` sh
python build_all.py -b TZ.BF.4.0 CHIPSET=sdm660 helloCA -c

python build_all.py -b TZ.BF.4.0 CHIPSET=sdm660 helloCA

#:<<!
adb shell mount /firmware -o rw,remount
# echo "start push smplap32...... goodixfp"
# find /home/yin/TZ.BF.4.0.7/trustzone_images/build/ms/bin/PIL_IMAGES/SPLITBINS_KAJAANAA/ -maxdepth 1 -name "smplap32.*" -exec adb -s f7a66acc push {} /firmware/image/ \;
# echo "push finished !" 
#cat /sys/kernel/debug/tzdbg/qsee_log

echo "start push helloCA......"
for f in /home/yin/Documents/TZ.BF.4.0.7/trustzone_images/build/ms/bin/PIL_IMAGES/SPLITBINS_KAJAANAA/helloCA.*
do
	echo $f
	adb -s f7a66acc push $f /firmware/image/
done
echo "push finished !"
#!
```

