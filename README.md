# 1. Install Ubuntu-20.4 on Host Machine

# 2. Check GPU's bus id at Host Machine
```
$ lspci -nn |grep -i nvidia
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107GL [Quadro P1000] [10de:1cb1] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
```
# 3. Edit /etc/default/grub at Host Machine
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX="quiet splash intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt vfio-pci.ids=10de:1cb1,10de:0fb9"
```
# 4. Update grub and Reboot at Host Machine
```
sudo update-grub
sudo reboot
```
# 5. Check VFIO at Host Machine
```
$ dmesg |grep -i vfio
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.8.0-59-generic root=UUID=33af4bcd-d3f9-4e6f-9ddd-5d6b2d02c044 ro quiet splash intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt vfio-pci.ids=10de:1cb1,10de:0fb9 quiet splash vt.handoff=7
[    0.046642] Kernel command line: BOOT_IMAGE=/boot/vmlinuz-5.8.0-59-generic root=UUID=33af4bcd-d3f9-4e6f-9ddd-5d6b2d02c044 ro quiet splash intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1 iommu=pt vfio-pci.ids=10de:1cb1,10de:0fb9 quiet splash vt.handoff=7
[    0.505185] VFIO - User Level meta-driver version: 0.3
[    0.505317] vfio-pci 0000:01:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none
[    0.523043] vfio_pci: add [10de:1cb1[ffffffff:ffffffff]] class 0x000000/00000000
[    0.543042] vfio_pci: add [10de:0fb9[ffffffff:ffffffff]] class 0x000000/00000000
[    2.238268] vfio-pci 0000:01:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none

$ lspci -nnk -d 10de:1cb1
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107GL [Quadro P1000] [10de:1cb1] (rev a1)
	Subsystem: NVIDIA Corporation GP107GL [Quadro P1000] [10de:11bc]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau

$ lspci -nnk -d 10de:0fb9
01:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
	Subsystem: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:11bc]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
```
# 6. Install KVM on Host Machine
```
$ sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
$ sudo apt install virt-manager
```
```
$ sudo vi /etc/security/limits.conf 
# qemu kvm, need high memlock to allocate mem for vga-passthrough
@kvm    hard    memlock 8388608
@kvm    soft    memlock 8388608
```
```
$ virt-install --name ubuntu2004 --memory 8192 --disk path=/mnt/kvm/ubuntu2004.img,size=30 --vcpus 2 --os-variant ubuntu20.04 --graphics none --console pty,target_type=serial --cdrom /mnt/os/ubuntu-20.04.2.0-desktop-amd64.iso --host-device=pci_0000_01_00_0 --machine q35
```
or Run virt-manager so that you can add the GPU into Guest Machine.
Select the VGA as Video. See attached files.
```
$ virt-manager
```
# 7. Install Ubuntu on Guest Machine and check the result lspci at Guest Machine
```
$ lspci -nn |grep -i nvidia
04:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107GL [Quadro P1000] [10de:1cb1] (rev a1)
05:00.0 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
$ ubuntu-drivers devices
$ sudo apt install nvidia-driver-460
$ reboot
$ nvidia-smi 
Unable to determine the device handle for GPU 0000:04:00.0: Unknown Error
```
# 8. Measures for "Unable to determine the device handle for GPU 0000:0x:00.0: Unknown Error" at Host Machine
```
$ virsh edit ubuntu20.04

Select an editor.  To change later, run 'select-editor'.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.tiny
  3. /bin/ed

Choose 1-3 [1]: 1
Domain ubuntu20.04 XML configuration edited.
```
Adding the followings:
```
・・・
  <features>
  ・・・
    <hyperv>
      <vendor_id state='on' value='whatever'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  ・・・
  </features>
・・・
```
```
$ nvidia-smi
Sun Jul 11 20:12:05 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 460.80       Driver Version: 460.80       CUDA Version: 11.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P1000        Off  | 00000000:04:00.0 Off |                  N/A |
| 34%   45C    P8    N/A /  N/A |     11MiB /  4040MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A       741      G   /usr/lib/xorg/Xorg                  4MiB |
|    0   N/A  N/A      1268      G   /usr/lib/xorg/Xorg                  4MiB |
+-----------------------------------------------------------------------------+
```
# 9. Additional tests at Guest Machine
you might be able to do below after "sudo apt-get install freeglut3-dev libglu1-mesa-dev".
```
vm01:/usr/local/cuda/samples/bin/x86_64/linux/release$ pwd
/usr/local/cuda/samples/bin/x86_64/linux/release

vm01:/usr/local/cuda/samples/1_Utilities/bandwidthTest$ sudo make
/usr/local/cuda-11.4/bin/nvcc -ccbin g++ -I../../common/inc  -m64    --threads 0 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_80,code=sm_80 -gencode arch=compute_86,code=sm_86 -gencode arch=compute_86,code=compute_86 -o bandwidthTest.o -c bandwidthTest.cu
nvcc warning : The 'compute_35', 'compute_37', 'compute_50', 'sm_35', 'sm_37' and 'sm_50' architectures are deprecated, and may be removed in a future release (Use -Wno-deprecated-gpu-targets to suppress warning).
/usr/local/cuda-11.4/bin/nvcc -ccbin g++   -m64      -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_80,code=sm_80 -gencode arch=compute_86,code=sm_86 -gencode arch=compute_86,code=compute_86 -o bandwidthTest bandwidthTest.o 
nvcc warning : The 'compute_35', 'compute_37', 'compute_50', 'sm_35', 'sm_37' and 'sm_50' architectures are deprecated, and may be removed in a future release (Use -Wno-deprecated-gpu-targets to suppress warning).
mkdir -p ../../bin/x86_64/linux/release
cp bandwidthTest ../../bin/x86_64/linux/release

vm01:/usr/local/cuda/samples/bin/x86_64/linux/release$ ./bandwidthTest 
[CUDA Bandwidth Test] - Starting...
Running on...

 Device 0: Quadro P1000
 Quick Mode

 Host to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)	Bandwidth(GB/s)
   32000000			12.8

 Device to Host Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)	Bandwidth(GB/s)
   32000000			13.0

 Device to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)	Bandwidth(GB/s)
   32000000			69.4

Result = PASS

NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.
```
```
vm01:/usr/local/cuda/samples/1_Utilities/deviceQuery$ sudo make
/usr/local/cuda-11.4/bin/nvcc -ccbin g++ -I../../common/inc  -m64    --threads 0 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_80,code=sm_80 -gencode arch=compute_86,code=sm_86 -gencode arch=compute_86,code=compute_86 -o deviceQuery.o -c deviceQuery.cpp
nvcc warning : The 'compute_35', 'compute_37', 'compute_50', 'sm_35', 'sm_37' and 'sm_50' architectures are deprecated, and may be removed in a future release (Use -Wno-deprecated-gpu-targets to suppress warning).
/usr/local/cuda-11.4/bin/nvcc -ccbin g++   -m64      -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_80,code=sm_80 -gencode arch=compute_86,code=sm_86 -gencode arch=compute_86,code=compute_86 -o deviceQuery deviceQuery.o 
nvcc warning : The 'compute_35', 'compute_37', 'compute_50', 'sm_35', 'sm_37' and 'sm_50' architectures are deprecated, and may be removed in a future release (Use -Wno-deprecated-gpu-targets to suppress warning).
mkdir -p ../../bin/x86_64/linux/release
cp deviceQuery ../../bin/x86_64/linux/release

vm01:/usr/local/cuda/samples/bin/x86_64/linux/release$ ./deviceQuery 
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "Quadro P1000"
  CUDA Driver Version / Runtime Version          11.4 / 11.4
  CUDA Capability Major/Minor version number:    6.1
  Total amount of global memory:                 4040 MBytes (4236312576 bytes)
  (005) Multiprocessors, (128) CUDA Cores/MP:    640 CUDA Cores
  GPU Max Clock rate:                            1481 MHz (1.48 GHz)
  Memory Clock rate:                             2505 Mhz
  Memory Bus Width:                              128-bit
  L2 Cache Size:                                 1048576 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total shared memory per multiprocessor:        98304 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 2 copy engine(s)
  Run time limit on kernels:                     Yes
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Managed Memory:                Yes
  Device supports Compute Preemption:            Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 4 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 11.4, CUDA Runtime Version = 11.4, NumDevs = 1
Result = PASS
```

# 10. Remove Swapfile (Optional)
```
$ sudo swapoff -v /swapfile
$ sudo vi /etc/fstab 
 comment out the line below:
 "/swapfile swap swap defaults 0 0"
$ sudo rm /swapfile
```

# 11. GPU testing
```
  $ sudo apt install nvidia-cuda-toolkit
  $ sudo apt install libssl-dev
  $ git clone https://github.com/developer-onizuka/vector.git
  $ ./gcc.sh
  $ time ./double_sqrt.co 134217728 |tail -n 5
  output:    5.000
  output:    5.000
  output:    5.000
  output:    5.000
  output:    5.000

  real	  0m38.429s
  user	  0m22.688s
  sys	  0m7.435s

  $ od -F -Ad  double_a.bin 
  0000000                        3                        3
  *
  1073741824
  $ od -F -Ad  double_b.bin 
  0000000                        4                        4
  *
  1073741824
  $ od -F -Ad  double_c.bin 
  0000000                        5                        5
  *
  1073741824
```
