# CUDAcons
Considerations on compiling [CUDA](https://de.wikipedia.org/wiki/CUDA) programs without a capable device (GPU). Short descriptions of findings gained while working on parallization of a Java application using [Parallel Java 2 Library](https://www.cs.rit.edu/~ark/pj2.shtml) (PJ2) and CUDA. Work was carried out on a Windows box with no CUDA device. A CUDA capable [AWS G2 instance](https://aws.amazon.com/de/blogs/aws/new-g2-instance-type-with-4x-more-gpu-power/) was used for testing.

### Compile CUDA programs on Windows without a capable device.
To compile CUDA programs one need a CUDA capable device (GPU), the device driver and the CUDA Toolkit with each dependent on it's predecessor. To compile programs that load kernels on CUDA devices Visual Studio needs to be in place as well. Take a look at [NVIDIA's documentation](https://docs.nvidia.com/cuda/cuda-c-programming-guide/) to get an in-depth understanding of developing CUDA programs. This section explores how far to come without a CUDA device.

Download and install [Cygwin](http://cygwin.com/) (x86_64) with GCC compiler suite. Consider a full install to avoid problems due to missing packages. Download [CUDA Toolkit](https://developer.nvidia.com/cuda-toolkit) and extract content (e.g. using [UniExtract](http://www.legroom.net/software/uniextract)).

#### To compile and link an executable...
...run commands in box below. This will compile one of the the CUDA samples that don't load a kernel. Resulting program should work on any CUDA capable machine. Compiler option `-static` assures that any GCC runtime libraries are *inside* the binary. Library referenced by options `-L$CUDA_HOME/lib/x64 -lcudart` is the *NVIDIA CUDA Runtime API* and a wrapper for `cudart64_75.dll` as seen in output of `nm $CUDA_HOME/lib/x64/cudart.lib`. The directory containing the DLL needs to be part of `PATH`. Check dynamic references with `ldd`.
```
# CUDA Toolkit 7.5
export CUDA_HOME=/usr/lab/cudacons/cuda_7.5.18_windows/CUDAToolkit
export PATH=$CUDA_HOME/bin:$PATH
# CUDA Toolkit 8
export CUDA_HOME=/usr/lab/cudacons/cuda_8.0.44_windows/compiler
export PATH=$CUDA_HOME/../cudart/bin:$PATH

x86_64-w64-mingw32-c++ \
	-I$CUDA_HOME/include \
	-I$CUDA_HOME/../CUDASamples/common/inc \
	-o deviceQuery.exe $CUDA_HOME/../CUDASamples/1_Utilities/deviceQuery/deviceQuery.cpp \
	-static -L$CUDA_HOME/lib/x64 -lcudart

ldd ./deviceQuery.exe
```

#### To build a dynamic link library (DLL)...
...run commands in box below (again no CUDA kernels). The example is from the [PJ2 distribution](pj2) which is solely for Linux. There is a shared object `libEduRitGpuCuda.so` in PJ2 that connects the Java side with CUDA. The illustrated example migrates the `.so` to a  `.dll`. The library referenced by options `-L$CUDA_HOME/lib/x64 -lcuda` is the *NVIDIA CUDA Driver API* and a wrapper for `nvcuda.dll` (see output of `nm $CUDA_HOME/lib/x64/cuda.lib`). The DLL is part of the NVIDIA driver package. It's name is `nvcuda64.dl_`. Expand file to `nvcuda.dll` and make sure that dynamic linker can find it. Run `ldd` to check references.
```
export JAVA_HOME=/cygdrive/c/program\ files/java/jdk1.7.0_71
export PJ2AWS_HOME=/usr/src/pj2aws
export PATH=$CUDA_HOME/../Display.Driver:$PATH

rm -f $CUDA_HOME/../Display.Driver/nvcuda.dll
/cygdrive/c/windows/system32/expand \
	`cygpath -w $CUDA_HOME/../Display.Driver/nvcuda64.dl_` \
	`cygpath -w $CUDA_HOME/../Display.Driver/nvcuda.dll`
# CUDA Toolkit 8
/cygdrive/c/windows/system32/expand \
	`cygpath -w $CUDA_HOME/../Display.Driver/nvfatbinaryloader64.dl_` \
	`cygpath -w $CUDA_HOME/../Display.Driver/nvfatbinaryloader.dll`
	
x86_64-w64-mingw32-gcc -Wall -shared \
	-I$PJ2AWS_HOME/pj2/lib \
	-I"$JAVA_HOME/include" \
	-I"$JAVA_HOME/include/win32" \
	-I$CUDA_HOME/include \
	-o EduRitGpuCuda.dll $PJ2AWS_HOME/pj2/lib/edu_rit_gpu_Cuda.c \
	-L$CUDA_HOME/lib/x64 -lcuda

ldd EduRitGpuCuda.dll
```

**Clue(s)**: **(a)** Check `PATH` if `ldd` won't show expected DLL's. **(b)** If `ldd` reports "???" instead of names run `strings <path> | grep -i dll` on each DLL file search output for DLLs and check if they exist and can be found. Continue recursively until "???" are gone.

#### To compile a CUDA kernel...
...install Visual Studio (e.g. Community edition). In case of VS 2015 use NVCC from CUDA Toolkit 8 (earlier versions of VS not examined). Start a Developer Command Prompt to get a propper VS environment and run NVCC on a `.cu` file.
```
rem turn off file name completion due to TABs in commands below.
cmd /f:off

set CUDA_HOME=%userprofile%\lab\cudacons\cuda_8.0.44_windows\compiler
set PATH=%CUDA_HOME%\bin;%PATH%

cd %userprofile%\lab\cudacons

nvcc -Wno-deprecated-gpu-targets -ptx ^
	-I%CUDA_HOME%\..\CUDASamples\common\inc ^
	-o clock.ptx %CUDA_HOME%\..\CUDASamples\0_Simple\clock\clock.cu
```

Maybe [LLVM can do the trick](http://llvm.org/docs/CompileCudaWithLLVM.html) as well...

### Compile CUDA programs on Linux without a capable device.
Almost the same as on Windows but without Visual Studio of course. Make sure GCC compiler suite and develoment tools are available. If on Red Hat Linux consider installation of *Develoment Tools*. Download runfile of CUDA toolkit and extract content.
```
# Install GNU C development tools
sudo yum groupinstall "Development Tools"

# Download CUDA runfile. Adjust URL as appropriate.
wget http://developer.download.nvidia.com/compute/cuda/7.5/Prod/local_installers/cuda_7.5.18_linux.run
# Unpack CUDA runfile
sh cuda_7.5.18_linux.run --extract=`pwd`
# Unpack CUDA driver in HOME directory
sh NVIDIA-Linux-x86_64-352.39.run --extract-only
# Install CUDA toolkit and samples
sudo sh cuda-linux64-rel-7.5.18-19867135.run
sh cuda-samples-linux-7.5.18-19867135.run
```

#### To compile and link an executable...
...run commands in box below. It should build one of the CUDA samples and run on any CUDA machine. Use `ldd` to check dynamic references.
```
export CUDA_HOME=/usr/local/cuda
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH

g++ -I$CUDA_HOME/include \
	-Icuda-samples-linux-7.5.18/common/inc \
	-o deviceQuery cuda-samples-linux-7.5.18/1_Utilities/deviceQuery/deviceQuery.cpp \
	-L$CUDA_HOME/lib64 -lcudart

ldd ./deviceQuery
```

#### To build a dynamic link library (DLL)...
...run commands in box below. As described above in the section on Windows this will build the native library from PJ2.
```
export JAVA_HOME=/usr/lib/jvm/java
export PJ2AWS_HOME=../pj2aws
export LD_LIBRARY_PATH=$PJ2AWS_HOME/pj2/lib:NVIDIA-Linux-x86_64-352.39:$LD_LIBRARY_PATH

( cd NVIDIA-Linux-x86_64-352.39 ; ln -s libcuda.so.352.39 libcuda.so )

gcc -Wall -shared -fPIC \
	-I$PJ2AWS_HOME/pj2/lib \
	-I$JAVA_HOME/include \
	-I$JAVA_HOME/include/linux \
	-I$CUDA_HOME/include \
	-o libEduRitGpuCuda.so $PJ2AWS_HOME/pj2/lib/edu_rit_gpu_Cuda.c \
	-LNVIDIA-Linux-x86_64-352.39 -lcuda

ldd libEduRitGpuCuda.so
```

#### To compile a CUDA kernel...
...add NVCC to `PATH` and run commands in box below.
```
export PATH=$CUDA_HOME/bin:$PATH

nvcc -Wno-deprecated-gpu-targets -ptx \
	-Icuda-samples-linux-7.5.18/common/inc \
	-o clock.ptx cuda-samples-linux-7.5.18/0_Simple/clock/clock.cu
```
