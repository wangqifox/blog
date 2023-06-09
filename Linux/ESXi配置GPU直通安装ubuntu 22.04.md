---
title: ESXi配置GPU直通安装ubuntu 22.04
date: 2023/05/24 15:48:00
---

在torch使用过程中，有时需要高版本的cuda。按照之前GPU直通的方式安装高版本驱动会出错，虚拟机设置`hypervisor.cpuid.v0 = "FALSE"`参数就会导致无法开机。索性安装一个高版本的ubuntu，配合安装cuda 11.8。

<!-- more -->

根据youtube的视频[https://www.youtube.com/watch?v=rhNCtsmVC30](https://www.youtube.com/watch?v=rhNCtsmVC30)整理了新的直通方案。

## 步骤1：将显卡设备切换为直通

点击“管理”→“硬件”→“PCI设备”。找到GPU相关的设备，将其切换为直通。

![esxi直通2204_1](media/esxi%E7%9B%B4%E9%80%9A1.png)

## 步骤2：配置机器

勾选预留所有客户机内存

![esxi直通2204_2](media/esxi%E7%9B%B4%E9%80%9A2204_2.png)

添加其他设备→PCI设备→添加GPU设备
![esxi直通2204_3](media/esxi%E7%9B%B4%E9%80%9A2204_3.png)

## 步骤3：安装ubuntu 22.04



## 步骤4：禁用nouveau驱动

1.使用下述命令可以查看 nouveau 驱动是否运行：

```Bash
lsmod | grep nouveau  
```

若出现下述结果：

```Bash
nouveau              1863680  9  
video                  49152  1 nouveau  
ttm                   102400  1 nouveau  
mxm_wmi                16384  1 nouveau  
drm_kms_helper        180224  1 nouveau  
drm                   479232  12 drm_kms_helper,ttm,nouveau  
i2c_algo_bit           16384  2 igb,nouveau  
wmi                    28672  4 intel_wmi_thunderbolt,wmi_bmof,mxm_wmi,nouveau  
```

说明 nouveau 驱动正在运行。

2.运行下述命令禁用该驱动：

```Bash
sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"  
sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"  
```

检查命令是否正确：

```Bash
cat /etc/modprobe.d/blacklist-nvidia-nouveau.conf  
```

若出现下述结果说明命令正确：

```Bash
blacklist nouveau  
options nouveau modeset=0  
```

3.更新设置并重启：

```Bash
sudo update-initramfs -u  
sudo reboot  
```

4.重启后重新输入下述命令：

```Bash
lsmod | grep nouveau  
```

若没有任何输出说明禁用 nouveau 驱动成功



## 步骤5：删除intel-microcode

执行

```Bash
sudo apt purge intel-microcode
sudo update-grub

```



执行`shutdown now`关机

## 步骤6：添加参数 

点击“编辑设置”→“虚拟机选项”→“高级”→“编辑配置”。添加参数：hypervisor.cpuid.v0 = "FALSE"。不添加这个参数，GPU驱动会检测到在虚拟机中运行，驱动就会不工作。 

![esxi直通2204_4](media/esxi%E7%9B%B4%E9%80%9A2204_4.png)

之后再开机

## 步骤7：安装GPU驱动

首先安装依赖：

```Bash
apt update
apt install gcc make build-essential libglvnd-dev pkg-config

```

安装驱动

```Bash
wget https://http.download.nvidia.com/XFree86/Linux-x86_64/520.56.06/NVIDIA-Linux-x86_64-520.56.06.run
chmod +x NVIDIA-Linux-x86_64-520.56.06.run
./NVIDIA-Linux-x86_64-520.56.06.run -m=kernel-open

```



或者直接安装cuda

```Bash
wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda_11.8.0_520.61.05_linux.run
chmod +x cuda_11.8.0_520.61.05_linux.run
./cuda_11.8.0_520.61.05_linux.run -m=kernel-open

```

编辑`/etc/profile`文件，添加：

```Bash
export PATH="/usr/local/cuda-11.8/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-11.8/lib64:$LD_LIBRARY_PATH"
```





安装完成之后编辑`/etc/modprobe.d/nvidia.conf`文件，添加：

```Bash
options nvidia NVreg_OpenRmEnableUnsupportedGpus=1
```

之后重启。之后执行`nvidia-smi`，可以看到驱动已经安装完成。

![esxi直通2204_5](media/esxi%E7%9B%B4%E9%80%9A2204_5.png)
