# DPU On ZC102
* 尝试在ZC102 Evaluation Board上烧录Linux系统的血泪文档
* 摘要
    * ZC102 Evaluation Board 
        * 核心芯片ZU9
        * ![](pics/board.jpg)
        * 通过SD卡进行启动
        * 外接显示器和鼠标键盘自备...
    * 开发环境
        * Vivado 2019.1 (版本限定)
        * petalinux 2019

## Environment

###  安装Vivado
1. 下载所需版本的Vivado安装文件。官方下载地址: [https://china.xilinx.com/support/download.html](https://china.xilinx.com/support/download.html)
2. 获取license(请各凭本事)。如果从官方购买了板卡，官方通常会提供一份license。
3. 运行安装程序，安装过程中根据需要选择版本以及所需组件(在硬盘空间不紧缺的情况下，建议选择System Edition版本，组件用默认选项即可)。安装时间较长。
4. 安装完成后，会自动进入Vivado License Manager(如果不小心关掉了，可以之后打开vivado，通过Help->License Manager进入)。选择Load License项，选择license文件。之后选择View License Status，看到可用的IP列表，即表示加载成功。
5. 在命令行执行以下命令将vivado添加到路径中，同时建议将该命令添加到~/.bashrc文件中以便命令行启动时自动添加该路径：
    ```bash
    source <path-to-vivado>/settings64.sh
    ```
6. 在命令行中运行以下命令开启Vivado:
   ```bash
   vivado &
   ```

* 注意：不同版本的Vivado并不冲突，只需要设置相应的路径即可完成切换。具有充足空间的服务器可以考虑保留多个版本的Vivado方便兼容不同的工程。

### 安装petalinux
参考资料：[UG1144](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2019_1/ug1144-petalinux-tools-reference-guide.pdf)
1. 下载所需版本的petalinux。petalinux的版本需要和Vivado的版本对应。官方下载地址: [https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools.html](https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools.html)
2. 根据UG1144中的说明安装所需的库。手册中会给出对应操作系统所需要运行的命令行代码。例如对于ubuntu：
   ```bash
   sudo apt-get install -y gcc git make net-tools libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget diffstat chrpath socat xterm autoconf libtool tar unzip texinfo zlib1g-dev gcc-multilib build-essential -dev zlib1g:i386 screen pax gzip
   ```
* 这里个人还需要再安装一个gawk库
    * 否则读取下载的install文件时会出错 
    ```
    Bad Address
    ```
3. 直接运行下载后的文件，在命令行中指定安装位置。注意：不能使用sudo权限或以root身份安装；安装的位置需要在当前用户的权限范围内。
   ```bash
   ./petalinux-v{xxxx.x}-final-installer.run <install-path>
   ```
* 启动安装文件时会报Warning
    ```
    WARNING: No tftp server found - please refer to "PetaLinux SDK Installation Guide" for its impact and solution
    ```
    * 查阅DataSheet(UG1144)后,本机没有开启tftp服务,会影响将linux镜像烧写到开发板,这里暂时可以不用考虑
        * 由于我们采用的SD卡启动,而不是通过TFTP进行烧写,所以关于这个的warning可以一律无视

* 这里可能的出错基本都是权限问题
    * 比如安装到/opt目录(如UG1144)用户可能没有权限(其实我是将/opt目录直接chmod成777的(有一点小危险),这样安装没有问题)
    * 还有安装文件(install script)不能设置为775权限(UG1144说这样的话bitBake会出问题),我目前是设置为777
    * 不能用ROOT安装!
    
4. 在命令行执行以下命令将petalinux添加到路径中，同时建议将该命令添加到~/.bashrc文件中以便命令行启动时自动添加该路径：
    ```bash
    source <path-to-petalinux>/settings.sh
    ```
    虽然不是一定要加,建议把这个也加上(后面如果遇到了找不到sourcing bitbake source一年的时候,有人给的解决方案是source一下这个(虽然我也不太确定这个有没有用,多source一个也花不了多少时间))
    ```bash
    source /opt/petalinux/components/yocto/source/aarch64/environment-setup-aarch64-xilinx-linux
    ```
* 将该文件添加到.bashrc时可能会导致开启bashrc较慢...属于正常现象(大概)
5. 运行以下命令验证配置是否成功，正确时返回安装路径：
   ```bash
   echo $PETALINUX
   ```

### 辅助软件及驱动
1. [CP210x USB转串口](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers)：用于通过串口连接ZCU102开发板。
2. [SD卡镜像烧写软件(Windows)](https://www.balena.io/etcher/)：烧写完整img格式镜像至SD卡。
3. SD卡分区软件(Linux)：SD卡格式化及分区。可以直接安装：
  ```bash
  sudo apt-get install gparted
  ```

## Vivado Set
* **注意**：以下部分相当于复现了Xilinx-DPU_TRD中的操作，可以直接在官网上下载trd，tcl脚本会自动完成配置，可以直接开始综合实现
1. 需要在Vivado(2019.1)下建立工程
2. 选择 Board(注意不是选芯片,而是选board,board Ver1.0,ver 3.3) -> ZC102
3. 需要首先在IP Catalog中导入DPU的IP
    * DPU的IP位于:
4. 建立Block Design (以下有超级多的工作,其实如果我的测试过了可以导出一个tcl)
    * 有以下几部分模块需要自行配置
        * Zynq UltraScale IP Core
            * 首先添加IP核,并且Run Block Automation(让Vivado帮你做一些初始化)
            * ![](pics/zynq0.png)
            * 然后做一些配置
            * ![](pics/zynq1.png)
            * ![](pics/zynq2.png)
            * ![](pics/zynq3.png)
            * 最后配置完的应该如以下所示
            * ![](pics/zynq4.png)
            * ![](pics/zynq5.png)  
            * 注意有没有多Port少Port,不然之后连线时会不好连
        * DPU IP Core
            * :cry:  我的DPU IP核里面显示是8h时限,过了之后可能需要regenerate(并无大碍)
            * 按照下图配置DPU核
            * ![](pics/DPU0.png)
            * ![](pics/DPU1.png)
            * ![](pics/DPU2.png)
            * 最终结果如下
            * ![](pics/DPU3.png)
        * Clock Part
            * 由一个clk_wizard和两个system reset构成,分别用来生成DPU所需要的DPU逻辑时钟(clk_dpu)和DSP时钟(clk_dsp)
                * clk_dsp: 650M 采用了DDR技术,获得两倍系统时钟的DSP时钟
                * clk_dpu: 系统时钟 325M
                * (Xilinx的工程师真的:cow::beer:,资源用满还能跑到325M)
                * 下面是配置:
                * ![](pics/clk0.png)
                * ![](pics/clk1.png) 
                * ![](pics/clk2.png) 这里记得要(拖下去)把rst type选成LOW
                * ![](pics/clk3.png) 这里也要把两个rst选成0,我当时被固定在1上,重启了一下Vivado就好了
        * Interrupt
            * 就是中断信号的concat...
            * ![](pics/intr0.png) 建立一个constant IP 
            * ![](pics/intr1.png) 
            * ![](pics/intr2.png)
        * AXI部分
            * PS(Zynq)与PL(DPU)通过AXI通信
            * ![](pics/axi.png) 就是例化一个AXI Interconnect,然后连线
    * DPU部分总框图:
        * ![](pics/DPU_all.png)
    * 完成了以上步骤之后,还需要在Address Editor(在block design的右边)那一栏,配置一下reg0的地址
        * ![](pics/addr_edit0.png)
        * ![](pics/addr_edit1.png)
        * (如果这一步不做的话会报超多的error和critical warning...) 
    * *如果采用了trd可以直接从这里开始*
5. 然后开始紧张刺激的Synth(30min) - Impl(1h) - Bitstream(<30min)吧!
    * 不出意外的话会报几百个warinng,但是没有critical warning就好...
    * ![](pics/synth0.png)
    * ![](pics/synth1.png)
6. File - Export Hardware - (勾选include bitstream)
    * 导出最后的 .hdf文件 (上面这么多步就是为了生成这么一个.hdf文件,包含了硬件信息和bit文件)

## PentaLinux Set

* 前面配置的时候有讲过,使用pentalinux需要source一个sh文件,可以放在~/.bashrc里面
    * (但是由于执行时间稍微有一些长,每次开bash都会卡一下子,所以我个人还是alias了一个指令,有需要的时候再用)
        * 可以把petalinux sh最后检查环境的代码删除，这样就快了
* 这一步的主要目标是产生系统镜像(包括image.ub和BOOT.bin)烧写到SD卡中让板卡启动
    * Petalinux 可以产生Rootfs,但是我们可以不用它的Rootfs

1. 建立工程
   ```bash
   petalinux-create -t project --name <project-name> --template zynqMP
   ```
   如果有板卡的BSP，可以采用BSP建立工程(会省很多配置)

   ```bash
   petalinux-create -t project -s <path-to-bsp>
   ```
   工程建立之后，会在当前路径下建立一个包含工程的文件夹。

2. 配置硬件及kernel（开发板可以选用默认选项不配置）
   ``` bash
   cd <path-to-project>
   petalinux-config --get-hw-descroption <path-to-folder-of-hdf>
   ```
   * petalinux默认的启动方式是initramfs，即将文件系统加载到内存中运行，而不会从SD卡中读取文件系统。因此需要单独配置启动方式。运行上面一条指令后会默认进入配置界面，需要配置以下内容：
   - 取消勾选```DTG Settings->Kernel Bootargs->generate boot args automatically```
   - 设置boot args为：```console=ttyPS0,115200 root=/dev/mmcblk0p2 rw earlyprintk rootfstype=ext4 rootwait```
        * bootarg是Linux中非常关键的一个环境变量
        * 一些常用的参数
            * root 指定rootfs的位置
            * rootfstype    
            * console
                * =tty 虚拟串口设备
            * mem 指定内存的大小
            * ramdisk([这篇博文](https://www.cnblogs.com/cornflower/archive/2010/03/27/1698279.html)表示最好用ramdisk_size) 表示ramdisk的大小，默认为4M
            * initrd/noinitrd (如果不使用ramdisk启动系统，需要noinitrd这个参数)
            * ip 系统启动之后的网卡地址(如果要使用基于nfs的文件系统，需要此参数)
            * Example
            ```
            setenv bootargs ‘initrd=0x32000000,0xa00000 root=/dev/ram0 console=ttySAC0 mem=64M init=/linuxrc’
            ```
        * 解释一下上面的Bootarg
            * console 使用虚拟串口tty0
            * root=/dev/mmcblk0p2 SD卡的第二分区
            * rw printk LOG输出
            * rootwait (rootdelay)用于boot时候文件系统可能还没有准备好的情况
   - 设置```Image Packaging Configurations->Root filesystem type```为```SD Card```

   * 配置完成之后
   需要更新现有的bootloader，保证bootloader最新
    ```
    petalinux-build -c bootloader -x distclean
    ```

   * 如果需要对内核进行其他配置，请根据[UG1144](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2019_1/ug1144-petalinux-tools-reference-guide.pdf)执行。

3. 配置Kernel
   ``` bash
   petalinux-config -c kernel
   ```
   然后需要配置一万个东西（带"//"的应该已经被BSP自动配置好了）
   ```
    Following are some of the mandatory configurations needed for successful booting of Ubuntu Desktop.
       Disable initramfs in kernel configuration GUI at ‘General setup -> Initial RAM file system and RAM disk (initramfs/initrd) support’
    Following settings are required to enable Input device, multimedia and USB related settings
       Device Drivers->Input device support->Event interface’　//
       Device Drivers->Input device support->Keyboards’        //
       Device Drivers->Input device support->Mouse interface’
       Device Drivers->Multimedia support->Media USB Adapters->USB Video Class (UVC)    //
       Device Drivers->Multimedia support->Cameras/video grabbers support’              //
       Device Drivers->Multimedia support->V4L platform devices                         //
       Device Drivers->USB support and enable all required classes (Except For Disable External Hub)
       Device Drivers->HID support->Generic HID driver                                      //
       Device Drivers->HID support->USB HID support->USB HID transport layer                //
    Disabling the PMBUS PMIC so that power demo can use them without any issues 
       Device Drivers->Hardware Monitoring support->PMBus support->Maxim MAX20751’  (Careful here is disable)
       Enable the PHY settings
       Device Drivers->PHY Subsystem’
       Device Drivers->PHY Subsystem->Xilinx ZynqMP PHY driver’
       Disable the PCI settings
       Bus Support->PCI support’ This needs to be disabled for this version
       Enable the sound related settings:
       Device Drivers->Sound card support’
       Device Drivers->Sound card support->Advanced Linux Sound Architecture’ enabling ALSA support
       Kernel hacking > Tracers > Kernel Function Tracer
    Save and Exit the kernel configuration.
   ```

   * rootfs同理 (这里config的是petalinux生成rootfs，不是我们最后用的那个，当然其实也可以用petalinux生成的rootfs,同样会生成在image/linux里面)
   ``` bash
   petalinux-config -c rootfs
   ```
        * // 这边petalinux-config时候,有时候可能会卡在(initializing...或者是ss check),原因是没有联网...
            * 解决方法: 输入 (单纯的config是可以config的,带别的指令的可能会卡住)
            ```
            petalinux-config
            ```
            向下滚动,到"Yocto Setting",取消勾选一个"Enable sstate feed from internet"(具体叫啥记不太清楚了,总之就是禁止联网校验)
            然后没有网也可以正常build和config啦!

        * // 这边如果卡在Sourcing Bitbake,参考上面安装流程中的
            ``` bash    
            source /opt/petalinux/components/yocto/source/aarch64/environment-setup-aarch64-xilinx-linux
            ```
            
4. Install DNNDK & DPU
    * 该部分[文档参考UG1144](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_3/ug1144-petalinux-tools-reference-guide.pdf),按照P77 - Building User Module章节
    * **配置设备树** 在完成正常的kernel config之后，修改设备树　将meta-user/recipes-bsp/device-tree/device-tree.bbappend和该目录下files/dpu.dtsi复制进去(TODO:可能还有一个带include的文件需要配置)
        * 如果设备树没有正确配置的话，系统将无法启动
        * petalinux-build之后系统文件夹下的dts与dtsi文件修改是没有用的,不是正确修改方法(虽然我第一反应是这么以为的)
    * **配置dpu驱动**  ./recepts-modules/dpu文件夹内容复制进去
        * 在复制之前，利用petalinux的指令加入该module　``` petalinux-create -t modules --name dpu --enable```
            * 如果在已经build过之后的文件夹下加入dpu module，会报错了``` Failed to open PetaLinux lib: librdi_commonxillic.so: cannot open shared object file: No such file or directory. ```解决方案，新建工程
    * **编译** 
        ```
        petalinux-build
        petalinux-build -c rootfs
        petalinux-build -x package
        petalinux-build  -c dpu       // (确保自建模块dpu被安装了)
        petalinux-build -c dpu -x do_insatll // (安装dpu模块)
        ```
        * 完成安装之后 在 %(PROJECT_DIRECTORY)/build/tmp/work/zcu102_zynqmp-xilinx-linux/目录下应有dpu/
    * **寻找并保存驱动文件** petalinux-build 完成之后，在build/tmp/work/zcu102_zynqmp-xilinx-linux/petalinux-user-image/1.0-r0/rootfs下搜索dpu.ko
        * 应该在lib/linux-x.xx/extra/dpu.ko，加入文件系统之后insmod一下


5. 编译（通常时间较长10-30分钟）
   ``` bash
   petalinux-build
   ```
   该过程会生成一系列文件，在```<path-to-project>/images/linux```文件夹内，我们最终需要用到的是image.ub(可能还有一个system.dts设备树)
   * image文件应该是在这一步骤产生的，如果我们按照默认的配置的话就是按照petalinux的默认配置dts和kernel选择
        * 当然前面如果调用过petalinux-config -c kernel的就使用的是我们配置的内核了

6. 打包镜像
   ``` bash
   petalinux-package --boot --format BIN \
       --fsbl   ./images/linux/zynqmp_fsbl.elf \
       --u-boot ./images/linux/u-boot.elf \
       --pmufw  ./images/linux/pmufw.elf \
       --fpga   ./images/linux/*.bit \
       --force
   ```
   * 这一步骤中各个文件的作用：
        * FSBL(1st Stage Bootloader)和zynq7000的FSBL相同，用CortexA53，也可以选择CortexR5来制作(取决于板卡的处理器型号)
        * pmufw.elf（所有的.elf其实是二进制可执行文件）MPSoc用来更好的管理电源和功耗，一般不需要修改
        * .bit PL的bitstream
        * uboot.elf uboot文件，一般来自于Xilinx官方提供，编译
        * image.ub 由kernel，rootfs和device tree打包来的

3. 以上步骤完成之后,会在images/linux/ 目录下产生BOOT.BIN和image.ub文件,这就是我们需要的

## Prepare SD Card
1. 第一使用SD卡时需要对SD卡进行格式化和分区。启动gparted，将SD卡分为两个分区：
   - BOOT分区：格式为FAT32，大小自定义，通常为几百MB，够用即可
   - ROOTFS分区：格式为EXT4，分配全部剩余空间即可
   结束之后记得把这两个分区mount上

    * 这里有时候会出现拔下读卡器之后无法umount分区的情况(提示device busy)
        ``` bash
        fuser -km '不能umount的分区所在的目录'
        sudo umount '不能umount的分区'
        ```

2. 将BOOT必要的文件拷贝到SD卡中
   ```bash
   cd <path-to-petalinux-project>/images/linux
   cp BOOT.BIN image.ub <path-to-BOOT-of-SDCARD>
   ```

3. 拷贝文件系统。可以使用```<path-to-petalinux-project>/images/linux```下的rootfs.tar.gz，也可以自己下载文件系统，例如从：[https://rcn-ee.com/rootfs/eewiki/minfs/](https://rcn-ee.com/rootfs/eewiki/minfs/)。或者是自己构建，之后将文件系统解压缩，运行如下命令：

   ```bash
   cd <path-to-rootfs>
   # 将文件系统复制到ROOTFS分区中，注意不可以使用cp
   sudo rsync -av ./ <path-to-ROOTFS-of-SDCARD>    
   # 清空缓冲区，确保文件写入SD卡
   sync
   # 修改权限，否则启动以后会出问题
   sudo chown root:root <path-to-ROOTFS-of-SDCARD>
   sudo chmod 755 <path-to-ROOTFS-of-SDCARD>
   ```
   * 该命令的执行地址要在文件系统的目录下面(注意不要选错了)
        * 有/bin /usr /home /bin /lib 的那个目录
        * 否则OS不能正确识别Rootfs文件系统,会报Kernel Panic的错误


## Board Setup
* 开发板配置好跳线设置(根据ZC102 DataSheet)
    * 确保其是从SD卡启动
* 将SD卡插入上电,通过DP连接到显示器
* 注意如果要使用外设(via USB2转USB3接口(图中那个形态怪异的转接器,板卡本身附带)),需要将J7短接
    * ![](pics/J7.jpg) 图中那个蓝色的跳线
* 因为内核配置问题,当不能使用键盘输入的时候,可以通过串口输入
    * 利用micro usb线连接靠近网口的那一个micro usb与电脑
    * 安装驱动后PC会识别出Silicon Bridge (4个串口)
        * 第0,1个是PS的串口: 我们需要观察的是**串口0**
            * 会打印一些调试信息(当系统起不起来的时候就靠它Debug...)
            * 当键盘外设没有配置成功的时候可以用来往里面输入(记住输指令的时候要把换行也输入进去...**MobaXTerm牛逼!流畅的调试！**)
        * 第2个是PL侧的串口
        * 第三个是板载的MSP430的串口

## Xilinx Example

* 按照Xilinx的最新版本的[教程](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841937/Zynq+UltraScale+MPSoC+Ubuntu+part+2+-+Building+and+Running+the+Ubuntu+Desktop+From+Sources)，采用2018.3的Vivado和Petalinux，按照这个教程，是可以正确运行的
* 唯一和教程中不大一样的几个点
    * 在内核编译的时候，有一项是选择usb driver下的proper part,除了"disbale hub"之外我全部选择了
    * 给的tcl文件并不会像它所说的能够直接生成hdf，而是要自己手动generate output products->create hdl wrapper之后export hardware
        * 这里不需要综合生成bit，输出的hdf file中描述了硬件资源和arm cpu的连接即可
* **最后的img会报很多错误...但是其实能起来，不用去管**
    * ```Unable to Load Kernel Module这种错误也属于正常...```
* **我在显示这里卡了好几天，尝试去解决很多boot时候的报错信息，但是他们其实都正常！！！实际出错的是hdmi->DP的线有问题！我用的是无源的转接器，那样不行！**
* 官方最后一次给的文件，能起 
    * ![](pics/xilinx_example.jpg)

## Self-Build Rootfs Start Desktop
* 到这一步你需要有的：
    * Ubuntu Rootfs:能联网apt-get
* 三种登录方式
    * 直接通过板子USB的键盘鼠标，和显示器登录
    * 从microusb登录
    * 通过ssh登录
* 有一些包是需要安装的　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　
    * lsmod -> apt-get install kmod
    * Network-Manager　（检查网络）
    * xinit　-> 包含了startx命令 (安装的时候会让你config一下键盘)
    * psmisc　->　包含了pstree(查看systemd的服务)
* 有一些config的相关文件需要复制过去
    * /etc/X11/xorg.conf.d/zynqmp.conf
    * driver文件
        * malixx.so    /usr/lib/x11/libmalixxx　（这个后来经过测试没有也可以）
        * armsoc_drv.so /usr/lib/xorg/modules/drivers/armsoc
    * *这一部分我当时是从官方的Example中直接复制出来的，后来才发现在官方的[meta-xilinx](https://github.com/Xilinx/meta-xilinx/tree/master/meta-xilinx-bsp/recipes-graphics)的repo中其实有*
* **开机手动Load　Arm Mali GPU驱动**
    * 复制mali.ko文件到本机
        * 并且在/etc/rc.local文件中添加一行　```insmod mali.ko```
#### 做到这一步是rootfs-9-1-base的状态
* 安装桌面环境
    * xfce4 ```sudo apt-get install xfce4```
        * 然后命令行登录的时候 ```startxfce4```
            * BUG1: 只能在root或是sudo的情况下进行登录
            * BUG2: 只能使用显示器默认分辨率，如果修改分辨率仅仅通过xrandr或者是图形界面里面的好像不行，可能需要修改zynqmp.conf文件并添加模式(仅测试过一次)
                * 参考rootfs-9-1-xfce4的zynqmp.conf文件，自己加了一个Monitor，并且设置了分辨率,这种方式设置应该开机默认就是1920x1080
            * BUG3: 该方式和xdm不兼容，若安装xdm(窗口管理器)并且开机自启动，登录之后会登录不上
    * ![](pics/xfce.jpg)
    * 在Xorg(图形界面的启动中，这样的一行错误属于正常现象)
        ```
        [   139.191] (EE) AIGLX error: dlopen of /usr/lib/aarch64-linux-gnu/dri/armsoc_d
        ri.so failed (/usr/lib/aarch64-linux-gnu/dri/armsoc_dri.so: cannot open shared oo
        bject file: No such file or directory)
        [   139.191] (EE) AIGLX: reverting to software rendering
        ```
    * 最后应当出现的现象：
        * 开发板启动从命令行登录，输入用户名和密码
        * 输入```sudo startx```或者```sudo startxfce4```
#### 这里备份为了rootfs-9-2

## Install ROS
* 本部分基本完全参考[该教程](https://github.com/xxzzll11111/SDcard_ROS_SPSLAM/blob/master/README.md),以及[该教程](https://sychaichangkun.gitbooks.io/ros-tutorial-icourse163/content/chapter1/1.4.html)
    * 该教程的系统环境是Debian9，我们的环境是Ubuntu16.04,需要做一定改变　melodic -> kinetic
* 注意一下要安装自己系统所对应版本的Ubnutu：
    * Ubuntu16.04 -> Kinetic　[*]
    * Ubnutu14.04 -> Indigo
    * Debian9 -> Melodic
1. 重装Opencv
    * 移除本身的OpenCV  ```sudo rm -r /usr/local/include/opencv2 /usr/local/include/opencv /usr/include/opencv /usr/include/opencv2 /usr/local/share/opencv /usr/local/share/OpenCV /usr/share/opencv /usr/share/OpenCV /usr/local/bin/opencv* /usr/local/lib/libopencv*```
    * 安装各种依赖　(经过自己测试都能过所以写成脚本一次性了)
        * 　./ros_install_dep.sh
    * 下载并解压OpenCV　from [这个链接](https://github.com/opencv/opencv/releases/tag/3.３.0)
    > 如果只是要安装ROS的话其实什么版本的Opencv都可以，但是如果使用DNNDK的话**一定**要安装其对应版本的Opencv　（DNNDK V2.08 - Opencv V3.1.0；　DNNDK V3.0 - Opencv V3.3.0（3.3.1都不行））
    * 为了正常的运行DPU提供的例子，还需要OpencvContrib的依赖，同样从github上[下载](https://github.com/opencv/opencv_contrib/releases/tag/3.3.0)(注意版本对应)，将其解压到一个位置
    * Build! （小破CPU需要跑个快15min）
    * ```
        cd opencv-3.1.0
        mkdir build
        cd build
        cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local -D ENABLE_PRECOMPILED_HEADERS=OFF -DOPENCV_EXTRA_MODULES_PATH=<opencv_contrib>modules　..
        make
        sudo make install
        sudo sh -c 'echo "/usr/local/lib" >> /etc/ld.so.conf.d/opencv.conf'
        sudo ldconfig
      ```  
    > 注意contrib_path的modules后面不要加 '/'
    * bash相关的配置 
    * ```
        sudo vim /etc/bash.bashrc
        # 在最末尾添加
        PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig 
        export PKG_CONFIG_PATH

        # 保存，执行如下命令使得配置生效
        source /etc/bash.bashrc

        # 更新
        sudo updatedb

        ```
2. 安装ROS
    * 添加sources.list; ppa-key并安装,最后的桌面环境有2G左右，需要安装好久
    ```
    sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.ustc.edu.cn/ros/ubuntu/ stretch main" > /etc/apt/sources.list.d/ros-latest.list'
    sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
    sudo apt-get update && sudo apt-get upgrade
    sudo apt-get install ros-kinetic-desktop-full # Ubuntu 16.04
    ``` 
    * 初始化 rosdep
    ```
    sudo rosdep init
    rosdep update
    ```
    * ROS环境配置
    ```
    echo "source /opt/ros/kinetic/setup.bash" >> ~/.bashrc
    source ~/.bashrc
    ``` 
    * ROS环境配置
    ```
    sudo apt-get install python-rosinstall python-rosinstall-generator python-wstool buid-essential
    ``` 
3. 测试环境(参考了[这个链接](https://blog.csdn.net/WY_star1/article/details/81564319))
    * Turtle demo
        * ```
          roscore &
          rosrun turtlesim turtlesim_node &
          rosrun turtlesim turtle_teleop_key & 
          ```
        * 用键盘移动乌龟吧!
#### 这里备份为了rootfs-9-３-withros

## Install DNNDK On Evaluation Board
* 参考[ug1327]()  P20
1. 在官网下载DPU所对应版本的DNNDK，解压之后将ZCU102/文件通过scp复制到板子上
2.  修改install.sh(删除安装dpu驱动以及覆盖BOOT.BIN以及设备树的部分)(参考../bak/install.sh)
    * 既然这里取消了，系统应该默认完成了insmod我们自己的dpu驱动
3.  运行install.sh应该能显示success
* 利用```dexplorer -v/w```来检测安装是否成功
    * 这里由于安装脚本时候用了sudo导致一些运行权限的问题 去/usr/locall/bin　把Permission Denied的对应命令给chmod了就好了
    * -v OUTPUT Is Like
    ```
    DNNDK version  3.0
    Copyright @ 2018-2019 Xilinx Inc. All Rights Reserved.

    DExplorer version 1.5
    Build Label: Apr 25 2019 09:50:06

    DSight version 1.4
    Build Label: Apr 25 2019 09:50:06

    N2Cube Core library version 2.3
    Build Label: Apr 25 2019 09:50:20

    DPU Driver version 2.2.0
    Build Label: Sep  3 2019 13:16:19
    ``` 
    * -w OUTPUT is like 
    ```
    [DPU IP Spec]
    IP  Timestamp   : 2019-09-03 10:45:00
    DPU Core Count  : 2

    [DPU Core List]
    DPU Core        : #0
    DPU Enabled     : Yes
    DPU Arch        : B4096F
    DPU Target      : v1.4.0
    DPU Freqency    : 325 MHz
    DPU Features    : Avg-Pooling, LeakyReLU/ReLU6, Depthwise Conv
    DPU Core        : #1
    DPU Enabled     : Yes
    DPU Arch        : B4096F
    DPU Target      : v1.4.0
    DPU Freqency    : 325 MHz
    DPU Features    : Avg-Pooling, LeakyReLU/ReLU6, Depthwise Conv

    [DPU Extension List]
    Extension Softmax
    Enabled         : Yes
    ``` 
4. 使用TRD中所包含的Demo Resnet50
* 从trd当中复制resnet文件夹(包括了可执行文件与pics)
* 执行可执行文件　resnet50
    * 可执行文件可能没有运行权限,chmod一下
```
cd $DIRECTORY_TO_EXEUCATBLE
./resnet50
```
* 到这一步骤理论上会报一个　```Error Loading Shared Lib: libopencv_core.so.3.1```
> 这并不是意思是需要Opencv.so.3.1,可能是一个奇妙错误，只要把它指向我们的libopencv_core.so.3.3.0就可以
```
cd /usr/local/lib
sudo ln -s libopencv_core.so.3.3.0 libopencv_core.so.3.1 // 创建一个libopencv_core.so.3.1指向libopencv_core.so.3.3.0
```
* 然后应该就能得到正确的结果了

# Trouble Shooting
* 系统起不起来，只显示到了Bootloader
    * 大概率是hdf文件的版本错了
* DP 出画面之后
    * Kernel Panic: 相当常见的错误,可能原因有很多
        * 检查Rootfs烧写进的文件夹位置是否正确
* 能够正常启动系统,但是串口中出来的log卡主
    * 并不是代表系统没起来，其实链接显示器并且使用键盘的话是可以正常登录的
    * 核心原因是挂载点没有包含正确的串口设备导致文件系统不能从串口登录
    * 参考了[这个帖子](https://forums.xilinx.com/t5/Embedded-Linux/Petalinux-2017-2-Ubuntu-RFS-on-Zynq-timed-out-waiting-for-device/td-p/800863)
    * 需要重新定向一下 ```ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttyPS0.service```
* 系统也能够正常启动了但是不能正常的链接网络
    * ifconfig出IP地址，eth0只有IPV6没有IPV4
    * 初步判断是DHCP没有work，可以改为手动配置静态IP，这个因人而异了
        * 可能是rootfs没有dncp module,首先配置静态IP让他能够连网
            * ```sudo apt-get install dhcpcd5```
    * (其实讲道理DHCP应该起身是能正常work的，我配置静态IP也没有其效果，最后换了一个网口就好了,原因未知)
* 系统启动之后登录的时候就报一个与权限有关的错误，然后发现自动补全的时候会报错 Couldn't create xxx **Read Only File System**
    * 我是更换了boot.bin但是没有更换rootfs，不知道对文件系统的权限做了啥导致它坏了
    * 在宿主机上首先将rootfs的区块umount（我的是/dev/sdd2）然后采用```fsck　/dev/sdd2```的方式修复(个人按了一万下y，本来以为铁定没用了，最后居然好像也许成功了？？　<-我是弱智，指令后面加-y)
        * 第二次操作发现没用...(靠难道是我build出来的东西不对？ - 好像我自己build出来的都会一开始有readonly file system)
        * 然后用了 ```mount -o remount rw ```正常了,加入文件系统的/etc/profile中，每次登录直接运行
    * 如果以上操作都没有用，检查SD卡的只读开关是否被移动了
* 在配置Xorg(图形界面)的过程中,ssh登录时间长,一段时间之后出现　　```xauth: “timeout in locking authority file /home//.Xauthority”```
    * 参考了[这个链接](https://blog.csdn.net/qq_39101111/article/details/78727843)的解决办法,别人说删除~/目录下与.Xauthority相关的文件
    * 这个好像就是Xorg没有配置正确的正常现象...稍等一会
