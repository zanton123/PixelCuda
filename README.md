#Compiling and running CUDA applications on the Google Pixel C using the GPUCC feature of clang, ptxas and fatbinary

One of the features for considering  a Pixel C is the NVIDIA Tegra X1 processor and its GPU that features 256 Maxwell CUDA cores. There are not many devices with the Tegra X1 in the portable category. By building a productivity device Google might cater here to the casual CUDA C++, PTX and MAXAS coder community that are now free to code away from home and internet. The device is potentially of high therapeutic value for CUDA addicts, for which there can be no cure obviously, but cannot be clinically certified until the Pixel C does CUDA.

As of writing, the Pixel C comes with Android Nougat 7.1.1, which includes CUDA 7.5. Oddly enough CUDA is somewhat hidden. The intention of this text is to assist with getting CUDA 7.0 running with small adaptations and some limitations (see below). You will be able to write, compile and run CUDA C++ code directly on the tablet without the need of any other computer. The strategy works in an Android terminal without root and has no graphical IDE.

##Install Termux by Fredrik Fornwall from the google play store
https://play.google.com/store/apps/details?id=com.termux

Running Termux opens an Android Terminal with a shell command prompt. This Android shell  features many commands known from Linux distributions. It also comes with a range of packages that have been cross-compiled from Linux to run on native Android, see more at: https://termux.com/

Follow the Termux online help to setup storage and preferences:
https://termux.com/help.html

##Building LLVM and clang with support for CUDA C++ and the NVPTX backend

Termux already comes with a package of clang 3.9.1, which will compile CUDA C++ to LLVM intermediate representation (IR) code but you will need the LLVM NVPTX backend to make PTX files. In addition the ptxas and fatbinary executables from the NVIDIA Linux for Tegra X1 toolkit need to be interfaced to run on Android to generate device code that can be embedded with your executables.

Information on compiling CUDA C++ with LLVM and the NVPTX backend are found at:
http://llvm.org/docs/CompileCudaWithLLVM.html
http://llvm.org/docs/NVPTXUsage.html

Install the following packages from the Termux shell:
apt install build-essential, cmake, make, perl, python2, wget

From the home/ folder:
* mkdir llvm3.9.1
* cd llvm3.9.1

Download LLVM and Clang source code version 3.9.1 from http://releases.llvm.org/download.html to your Download folder using your Web browser then move the files into the current home/llvm3.9.1/ folder and unpack the source:
* mv ~/storage/shared/Download/llvm-3.9.1.src.tar.gz ./
* mv ~/storage/shared/Download/cfe-3.9.1.src.tar.gz ./
* tar -xvJf llvm.tar.xz
* tar -xvJf clang.tar.xz

Move the unpacked clang source folder cfe-3.9.1.src/ into llvm.src/tools/ folder:
* mv cfe-3.9.1.src llvm-3.9.1.src/tools/

Enter clang.src/ and set environment variable to source root directory for convenience
*cd llvm-3.9.1.src
* SRC_ROOT=*`pwd`*

(echo $SRC_ROOT should return /data/data/com.termux/files/home/llvm3.9.1/llvm-3.9.1.src)

Now you need a build folder where llvm executables will be put outside of the source folder:
* cd ~/llvm3.9.1
* mkdir build

Enter the ~/llvm3.9.1.src/build/ folder and build llvm with the Aarch64 and NVPTX targets:
* cd build
* cmake -G “Unix Makefiles” -DMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=prefix=/data/data/com.termux/files/usr/llvm -DLLVM_TARGETS_TO_BUILD=”Aarch64;NVPTX” -DCMAKE_C_COMPILER=/data/data/com.termux/files/usr/bin/cc -DCMAKE_CXXCOMPILER=/data/data/com.termux/files/usr/bin/c++ -DCMAKE_ASM_COMPILER=/data/data/com.termux/files/usr/bin/as $SRC_ROOT/
* make

Most of llvm compiles in Termux but there seems to be a missing definition of the std::to_string() method in Android which makes llvm3.9.1.src/tools/sancov/sancov.cc fail in line 512. If the error occurs you can edit cancov.cc to insert a workaround using the c++ stringstream API.

* vi ../llvm3.9.1.src/tools/sancov.cc

In the editor press i for insert mode and add a new line 53:
* #include <sstream> // try to get around std::to_string() ERROR with construct line 512 ff

In the implementation of the function formatHtmlPct() in source line 512 substitute:

    std::string Num=std::to_string(Pct);

with:

    std::string Num;
    std::stringstream ss_Num;
    ss_Num << Pct;
    Num = ss_Num.str();
    // substitute for Android error in std::string Num = std::to_string(Pct);

Press ESC or Ctrl+c followed by :wq ENTER to write and close the file and continue building:
* make

This time the entire build of llvm should succeed and the executables can be added to the path:
* export PATH=/data/.../home/llvm3.9.1/build/bin:$PATH

To set the PATH in Termux at startup, add above line to the ~/.bashrc shell script. However, this will change your compiler system wide and could interfere with Termux at some point. It is a better idea to make a separate shell script to set the path for CUDA development and before this store the current Termux PATH in a script to revert:
* echo $PATH > normal
* echo "export PATH=~/llvm3.9.1/build/bin:$PATH" > ~/cuda

Use source to invoke the script to switch to CUDA development:
* . ~/cuda

... and revert back to normal:
* . ~/normal


##NVIDIA CUDA for Android

Nvidia provides support for CUDA development for Android in their CodeWorks for Android toolkit:
https://developer.nvidia.com/codeworks-android

The CUDA toolkit includes several CUDA libraries, of which the CUDA runtime library libcudart.so is most important for running executables with embedded CUDA kernels. Most other libraries provide extended functionality including Fourier transformation, linear algebra and others. Generally, CUDA driver API applications link to libcuda.so, which is supplied by the CUDA driver that is installed with the system and in case of the Pixel C cannot be changed unless the tablet is rooted. Thankfully, the Pixel C comes with a GPU Android kernel module and libnvcompute.so, which is the Pixel C version of the CUDA driver library. For CUDA runntime API applications it appears that libcudart.so dynamically loads libcuda.so (not listed as dynamic dependency in the ELF), which then interfaces with the GPU Android/Linux kernel module. (Linux/Android kernel and CUDA kernels are different). On the Pixel C libnvcompute.so can be used instead of libcuda.so. The facility for building a complete CUDA runtime and driver API has been pointed out some time ago by Amy Wong in a tweet storm from Marsvegas answering to desperate request for CUDA on the Pixel C:

https://devtalk.nvidia.com/default/topic/930672/pixel-c-access-to-cuda-/

The situation went a bit out out of hand here. One might call it contrived thinking that the NVIDIA Tegra X1 development kit would look a bit odd and the Pixel C would look so sleek when carried around, as this obviously ignores what a CUDA addict would look like with a Pixel C and no access to CUDA. Amy's post, brief and concise though maybe not explicitly detailed, is the answer to the final question:

**"Anyone has method can make Pixel C supported with CUDA access ? "**  '42'
 
The newest version of NVIDIA CodeWorks for Android 1R5 now provides support for Android Marshmallow and CUDA 7.0 for Tegra X1 devices and is compatible with the higher versions of the Google Pixel C. NVIDIA CodeWorks needs to be installed on an Intel/AMD x64 processor bearing PC running Ubuntu 14.04 (AMD64). Point your Web browser to:

https://developer.nvidia.com/gameworksdownload#?dn=codeworks-for-android-1r5

and select Codeworks for Android 1R5 2016/09/07 DOWNLOADS Ubuntu (64 bit) in your Web browser. You will be asked to sign into your NVIDIA Developer program account. If not a member yet, it is time to register with NVIDIA and signup, which is usually quick. You can then download the CodeWorksforAndroid-1R5-linux-x64.run file. Locate the file in the Downloads folder and add execute permission and run in a Ubuntu terminal (not on the Pixel C):

cd Downloads/
chmod +x CodeWorksforAndroid-1R5-linux-x64.run
./CodeWorksforAndroid-1R5-linux-x64.run

This will install NVIDIA's Android Studio and the CUDA toolkit. If you are runnuing Ubuntu on an old Laptop that has no CUDA installed, the CUDA toolkit might not install automatically with CodeWorks. It is still possible to get the CUDA toolkit by first installing an older version of AndroidWorks or GameWorks for Android and then upgrading to CodeWorks for Android 1R5 with above run file.

After the installation you should have a full development environment for Android including an Eclipse based IDE setup for native Android development. For our purpose the files in the ~/NVPACK/cuda-android-7.0/aarch64-linux-androideabi/ folder are of interest. The include/ folder contains the CUDA header files, the lib64/ folder contains the CUDA libraries including libcudart.so for Android, the samples/ folder with the CUDA samples, and the bin/ folder contains some useful tools for profiling and analysing CUDA applications on Android. Transfer these folders to a portable disk or USB drive.
In Termux navidate to the usr/ folder and add a local/cuda-7.0/ folder:
cd ~/../usr
mkdir local
cd local
mkdir cuda-7.0

Add a symbolic link to cuda:
ln -s cuda-7.0 cuda

Copy the files from your portable drive to the Download folder using a file manager App (eg. ES File Explorer). Then copy the files and all subfolders to the usr/local/cuda-7.0/ folder:
cd cuda-7.0
mkdir bin
cp -r ~/storage/shared/Downloads/cuda-android-7.0/aarch64-linux-androideabi/bin/* bin/
mkdir include
cp -r ~/storage/shared/Downloads/cuda-android-7.0/aarch64-linux-androideabi/include/* include/
mkdir lib64
cp -r ~/storage/shared/Downloads/cuda-android-7.0/aarch64-linux-androideabi/lib64/* lib64/
mkdir samples
cp -r ~/storage/shared/Downloads/cuda-android-7.0/aarch64-linux-androideabi/samples/* samples/

Now add a symbolic link for lib and libcuda.so:
ln -s lib64/ lib
cd lib64
ln -s /system/vendor64/libnvcompute.so libcuda.so

The NVIDIA CUDA for Android files are now in /data/data/com.termux/files/usr/local/cuda-7.0 and we have a clang for compiling CUDA C++. What is still missing are the NVIDIA tools ptxas and fatbinary for generating device code that can be embedded in executables. However, with the present installation you can generate ptx files that can be loaded using the CUDA driver API (see the vectorAddDrv or matrixMulDrv CUDA samples). It is a good idea to test for functionality at this point. We need to set the LD_LIBRARY_PATH, which currently points to /data/data/com.termux/usr/lib (use echo $LD_LIBRARY_PATH). For this we add to our ~/cuda and ~/normal shell scripts:
echo "LD_LIBRARY_PATH=$LD_LIBRARY_PATH" >> ~/normal
echo "export LD_LIBRARY_PATH=/system/lib64:/data/data/com.termux/files/usr/lib:/data/data/com.termux/files/local/cuda/lib64:/system/vendor/lib64" >> ~/cuda

Move the cuda samples into the home folder:
cd ~
mkdir CUDA-7.0-samples
mv ../usr/local/cuda/samples/ CUDA-7.0-samples/

First try to compile the deviceQuery sample:
cd 1_Utilities/deviceQuery
. ~/cuda
gcc --verbose --cuda-path=/data/data/com.termux/files/usr/local/cuda --cuda-gpu-arch=sm_53 -I../../../../usr/local/cuda/include -I../../common/inc -lgnustl_shared -L /data/data/com.termux/files/usr/local/cuda/lib64 -lcudart -L /sytsem/lib64 -L /system/vendor/lib64 -lnvcompute -o deviceQuery deviceQuery.cpp

This should compile and give an executable deviceQuery in the same folder. To run type:
./deviceQuery

You should see no a list of capabilities of your Tegra X1 Maxwell GPU. Next navigate to the vectorAddDrv example.
cd ../../0_Simple/vectorAddDrv/

Compile vectorAdd_kernel.cu with the clang compiler we build before (ignore linker warning):
clang++ -target aarch64-linux-android --cuda-path=/data/data/com.termux/files/usr/local/cuda --cuda-gpu-arch=sm_35 ../../../../usr/local/cuda/include -I../../common/inc -I../../../../usr/include --cuda-device-only -S -o vectorAdd_kernel64.ptx  vectorAdd_kernel.cu

A ptx file vectorAdd_kernel64.ptx should now be in the same directors. Use cat vectorAdd_kernel64.ptx to see the ptx code. Next compile the vectorAddDrv.cpp using the compiler comming with Termux gcc (clang 3.9.1):
gcc --cuda-path=/data/data/com.termux/files/usr/local/cuda --cuda-gpu-arch=sm_53 -I../../../../usr/local/cuda/include -I../../common/inc -lgnustl_shared -L /data/data/com.termux/files/usr/local/cuda/lib64 -lcudart -L /system/lib64 -L /system/vendor/lib64 -lnvcompute -o vectorAddDrv vectorAddDrv.cpp

You can run the vectorAddDrv executable in the same folder with:
./vectorAddDrv

If the output ends with "Result = PASS" everything looks promising. When troubleshooting consider that we used the compiler that comes with Termux for the .cpp files, and only the .ptx file was generated by the clang version built from source. Errors can be related to either PATH, LD_LIBRARY_PATH, missing files or the Termux clang compiler setup (gcc). You can also use the Termux compiler the generate LLVM IR code from .cu files and then use the NVPTX backend llc from the LLVM version built from source to get ptx code:
g++ -s --cuda-path=/data/data/com.termux/files/usr/local/cuda --cuda-device-only -emit-llvm --cuda-gpu-arch=sm_53 -I../../../../usr/local/cuda/include -I../../common/inc vectorAdd_kernel.cu -o vectorAdd_kernel.bc

vectorAdd_kernel.bc LLVM IR byte code can be passed to the NVPTX backend:
llc -mtriple=nvptx64-nvidia-cuda -filetype=asm vectorAdd_kernel.bc -o vectorAdd_kernel64.ptx


##NVIDIA Linux for Tegra

We are presently stuck at PTX and cannot go any further with open source tools. NVIDIA also does not supply CUDA compiler components that run natively on Android (the toolkit contains onyl profiling and analysis tools ported to Android). However, there are ARM64 versions for the Tegra X1 in the Linux for Tegra Archive: 

https://developer.nvidia.com/embedded/linux-tegra-archive

From the features Jetson TX1 R24.1 – May 2016 contains the CUDA 7.0 tools matching our CUDA installation and can be downloaded from:

https://developer.nvidia.com/embedded/downloads

Scroll down the list and select JetPack for L4T 2.2 2016/06/15 for download on your Ubuntu PC. Locate JetPack-L4T-2.2-linux-x64.run in an Ubuntu terminal and run:
cd ~/Downloads
chmod +x JetPack-L4T-2.2-linux-x64.run
./JetPack-L4T-2.2-linux-x64.run

When prompted select JETPACK 2.2 – Tegra X1 64 bit. The installer will download all files and then ask to flash the operating system image to a Jetson X1 development board. At this point the installation can be aborted. In the ~/Downloads/Jetpack_2.2/jetpack_download/ folder two archives cuda-repo-l4t-7-0-local_7.0-76_arm64.deb contains the ARM64 CUDA 7.0 toolkit and Tegra_Linux_Sample-Root-Filesystem_R24.1.0_aarch64.tbz2 contains the ARM64 Linux root file system. Use Ubuntu Archive Manager to open these archives and the use drag and drop in the file manager window to extract and copy files and folders from these archieves (on the file right click the mouse to see the options for open).

From the Tegra_Linux_Sample-Root-Filesystem_R24.1.0_aarch64.tbz2 archive copy the following files from the /lib/aarch64-linux-gnu/ to a L4T-files/ folder on a portable drive:

ld-2.19.so
libc-2.19.so
libm-2.19.so
libdl-2.19.so
libpthread-2.19.so
libz.so.1.2.8

Copy the L4T-files/ folder to the Download folder on the Pixel C and move them to a new /data/data/com.termux/files/usr/lib64/ folder (to separate Linux and Android binaries):
cd ~/../usr
mkdir lib64
cp ~/storage/shared/Download/L4T-files/* lib64/

Make symbolic links in the same folder (ie lib64/) as follows:
cd lib64
ln -s ld-2.19.so ld-linux-aarch64.so.1
ln -s libc-2.19.so libc.so.6
ln -s libm-2.19.so libm.so.6
ln -s libdl-2.19.so libdl.so.2
ln -s libpthread-2.19.so libpthread.so.0
ln -s libz.so.1.2.8 libz.so.1

Add execute permission to the Linux dynamic linker loader:
cd lib64
chmod +x ld-2.19.so

The ld-2.19.so dynamic loader and the shared libaries are required to run the Linux 64 bit versions of ptxas and fatbinary which are in the cuda-repo-l4t-7-0-local_7.0-76_arm64.deb archive. Extract /var/cuda-repo-7-0-local/cuda-core-7.0-76_aarch64.deb and copy the /usr/local/cuda-7.0/ folder to a portable drive and then into the Download folder on the Pixel C. 
Navidate to the cuda-7.0 folder on the Pixel C and copy the bin/ and nvvm/ folders:
cd ~/../usr/local/cuda-7.0
mkdir nvvm
cp -r ~/storage/shared/Downloads/L4T-cuda/bin/* bin/
cp -r ~/storage/shared/Downloads/L4T-cuda/nvvm/* nvvm/ 

To run ptxas and fatbinary executables for Linux on Android we simply make shell scripts in the /data/data/com.termux/files/usr/bin/ folder using the vim editor:
cd ~/../usr/bin
vi ptxas

Enter insert mode (i) write the following lines in the editor and close (Ctrl+c :wq ENTER): 

#!/data/data/com.termux/files/usr/bin/sh
/data/data/com.termux/files/usr/lib64/ld-linux-aarch64.so.1 /data/data/com.termux/files/usr/local/cuda-7.0/bin/ptxas $*

vi fatbinary

Enter insert mode (i) write the following lines in the editor and close (Ctrl+c :wq ENTER): 

#!/data/data/com.termux/files/usr/bin/sh
/data/data/com.termux/files/usr/lib64/ld-linux-aarch64.so.1 /data/data/com.termux/files/usr/local/cuda-7.0/bin/fatbinary $*

To make it work we also need to add the usr/lib64 folder to the LD_LIBRARY_PATH.

vi ~/cuda

In the editor (i) add :/data/data/com.termux/files/usr/lib64 to the LD_LIBRARY_PATH which then should read as follows:
export LD_LIBRARY_PATH=/system/lib64:/data/data/com.termux/files/usr/lib:/data/data/com.termux/files/local/cuda/lib64:/system/vendor/lib64:/data/data/com.termux/files/usr/lib64

Then close the file with Ctrl+c :wq ENTER. 

Note: There is an incompatibility between /data/data/com.termux/files/usr/lib/libunwind.so and /system/lib64/libunwind.so. As libnvcompute.so depends on the version in /system/lib64 we set this first in the LD_LIBRARY_PATH. LD_LIBRARY_PATH is switched back with the ~/normal shell script for running apt and other Termux executables.

Now it is time to test a real CUDA runtime executable. For this enter the vectorAdd sample folder, compile and run the sample:
cd ~/CUDA-7.0-samples/0_Simple/vectorAdd
. ~/cuda
clang++ -target aarch64-linux-android --cuda-path=/data/data/com.termux/files/usr/local/cuda --cuda-gpu-arch=sm_35 -fPIC -pie -I../../../../usr/local/cuda/include -I../../common/inc -I../../../../usr/include --sysroot=/data/data/com.termux/files/usr -L /system/lib64 -L /system/verdor/lib64 -lnvcompute -lgnustl_shared _L /data/data/com.termux/files/usr/local/cuda/lib64 -lcudart -o vectorAdd vectorAdd.cu -v

An executable file vectorAdd should have been generated that can be run with:

./vectorAdd

An output ending with "Test PASSED Done" is an encouraging sign and you can take it from there. Good luck! The Pixel C CUDA strategy would have been all but impossible if not for Amy Wong, Fredrik Fornwall, and visionary teams at NVIDIA and Google. Thanks!


##Limitations

###CUDA 7.5 on the Pixel C
CUDA 7.0 is the latest version of libcudart.so in the NVIDIA Codeworks for Android toolkit. We  need a CUDA runtime library for Android and  cannot use a newer versions from the Linux for Tegra development kit. Therefore the device code NVVM library is only available up to Compute Capability 3.5 (Tegra X1 has Compute Capability 5.3) and 16 bit floats are missing.

###32 bit armv7l CUDA applications on the Pixel C
Google Pixel C has support for armv7l libraries including a 32-bit version of libnvcompute.so in /system/vendor/lib, which can be used to run armv7l CUDA applications. Compiling 32-bit CUDA requires the armv7l libraries from the Codeworks for Android CUDA development kit, changes to the clang command line arguments, and setting LD_LIBRARY_PATH to point at the /system/lib;/system/vendor/lib folders.

###Cuda streams other then the default stream do not work for kernel execution
Compiling of the concurrentKernels.cu sample works. However, it appears that streams in the kernel launch command are not working. Whereas streams can be specified for memory copies, adding a stream parameter to the <<<..., cuda_stream>>> command appears to make the kernel not run. Oddly the kernel launch passes silently generating neither an error nor any output. This is a limitation that could be related to the implementation in libnvcompute.so and likely will not be corrected as there is no official CUDA support. Preliminary tests indicate that this limitation (it is not a bug as there is no official CUDA on the Pixel C) is independent of LLVM and also occurs with nvcc (prooted from the L4T version from Jetpack 2.2; CUDA 7.0).
check 32 bit / check on Shield Tablet

###CUDA sample matrixMul has a long execution time or hangs
Compiling the matrixMul.cu sample works. However, executing matrixMul has a long runtime or hangs. This appears a problem with LLVM as the same code compiled with nvcc (prooted from the L4T from Jetpack 2.2; CUDA 7.0) runs normally. Investigating the problem shows that the program hangs at cudaDeviceSynchronize() after the first kernel launch in the warm up section completes apparently normally. It is not clear, what the reason for this is but could be related to specifics of Tegra devices and the memory space that GPU and CPU share. Notably, this CUDA sample relies on a large number of kernel launches. LLVM handle the reuse of kernels or the CUDA cache differently than nvcc. However, the CUDA cache is handled by the driver and the problem clearly occurs before kernel relaunch making this scenario rather unlikely.
