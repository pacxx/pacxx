## Welcome to PACXX! ##
 [![pipeline status](https://zivgitlab.uni-muenster.de/HPC2SE-Project/pacxx-llvm/badges/master/pipeline.svg)](https://zivgitlab.uni-muenster.de/HPC2SE-Project/pacxx-llvm/commits/master)

The PACXX (Programming Accelerators with C++) Project started in 2013 as PhD Thesis and is finally open source. 

PACXX is a simple, lightweight and still powerful programming model for accelerators in C++. PACXX was primary planed as replacement to CUDA in a time without C++11/14 support. In the past years PACXX did not only advance GPU programming to C++14 and beyond, but also becomes portable across different hardware architectures. 

Currently, PACXX supports Nvidia GPUs with Compute Capability of 2.0 and above, CPUs from different vendors (Intel, AMD, ARM) and in some weeks from now PACXX will rock on ROCm enabled GPUs from AMD as well. 

# Dependencies 

# Getting Started

 First of all clone the source:

 The convient way is to use **repo**. You can get **repo** here.

 
``` bash
repo init -u git@github.com:pacxx/pacxx.git
repo sync
```

Repo will set up all sub reporsitories for you to get going including the PACXX runtime and the modified Clang frontend. 

 Build PACXX: 

``` bash
mkdir build && cd build

cmake .. -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=ON -DLLVM_ENABLE_RTTI=ON -DLLVM_ENABLE_CXX1Y=ON -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS_RELEASE="-O3" -DCMAKE_INSTALL_PREFIX=<some_path>

make -j<number of cores>
make install 
```

Get some coffee, this can take some time.

# Write your first PACXX Program

``` C++
#include <PACXX.h>
#include <vector>
#include <algorithm>

using namespace pacxx::v2;

int main(int argc, char *argv[]) {

  Executor::Create<CUDARuntime>(0); // create an executor

  auto &exec = Executor::get(0);    // retrieve the default executor

  size_t size = 128;

  std::vector<int> a(size, 1);      // allocate some memory on the host
  std::vector<int> b(size, 2);
  std::vector<int> c(size, 0);
  std::vector<int> gold(size, 0);

  auto &da = exec.allocate<int>(a.size());  // allocate some memory on the device 
  auto &db = exec.allocate<int>(b.size());
  auto &dc = exec.allocate<int>(c.size());

  da.upload(a.data(), a.size());    // upload data to the device
  db.upload(b.data(), b.size());
  dc.upload(c.data(), c.size());

  auto pa = da.get();   // grab the raw pointer from the device address space
  auto pb = db.get();
  auto pc = dc.get();

  auto vadd = [=](range &config) {  // define the vector addition kernel
    auto i = config.get_global(0);  // get the global id (in x-dimension) for the thread  
    if (i < size)
      pc[i] = pa[i] + pb[i] + 2;
  };

  exec.launch(vadd, {{1}, {128}});  // launch the kernel with 128 threads in 1 block
  dc.download(c.data(), c.size());  // download the results from the device 

  std::transform(a.begin(), a.end(), b.begin(), gold.begin(), [](auto a, auto b) { return a + b + 2; }); 
  if (std::equal(c.begin(), c.end(), gold.begin())) // check the results
    return 0; // passed
  else
    return 1; // failed
}
```

To compile your code, PACXX comes with a handy compiler driver called `pacxx++`  which will compile your code and may also be a valid CXX compiler if you work with build tools like cmake:

Let's say you wrote the code above and stored it as `vadd.cpp` on your disk:

```bash
pacxx++ -O3 vadd.cpp -o vadd
```

is enough to get you the executable you wanted.
If everything was set up correctly you should now get an executable linked against the PACXX runtime and you are good to go. 

Running the executable with `PACXX_LOG_LEVEL=2` env variable set will give you the verbose output of the runtime: 

```
CUDARuntime.cpp:186: note: VERBOSE: CUDARuntime has found 1 CUDA devices
CUDARuntime.cpp:34: note: VERBOSE: Creating cudaCtx for device: 0 0 0x2055550
CUDARuntime.cpp:43: note: VERBOSE: Initializing PTXBackend for Tesla K20c (dev: 0) with compute capability 3.5
PTXBackend.cpp:52: note: VERBOSE: Intializing LLVM components for PTX generation!
CoreInitializer.cpp:32: note: VERBOSE: Core components initialized!
Executor.cpp:88: note: VERBOSE: Created new Executor with id: 0
MSPEngine.cpp:49: note: DEBUG: MSP Engine disabled!
CUDARuntime.cpp:93: note: VERBOSE: //
                                   // Generated by LLVM NVPTX Back-End
                                   //
                                   
                                   // ptx stripped for shortness                  
Timing.h:45: note: VERBOSE: CUDARuntime.cpp:71 compileAndLink timed: 4497us
Executor.h:191: note: VERBOSE: allocating memory: 512
Executor.h:191: note: VERBOSE: allocating memory: 512
Executor.h:191: note: VERBOSE: allocating memory: 512
CUDAKernel.cpp:43: note: VERBOSE: setting kernel arguments
CUDAKernel.cpp:51: note: DEBUG: Launching kernel: _ZN5pacxx2v213genericKernelIZL19test_vadd_low_leveliPPcE3$_0EEvT_
CUDAKernel.cpp:55: note: VERBOSE: Kernel configuration: 
                                  blocks(1,1,1)
                                  threads(128,1,1)
                                  shared_mem=0
Executor.h:99: note: VERBOSE: destroying executor 0
```
# Known Issues

- [ ] Atomic Operations are more or less a bad hack.
- [ ] Missing support for constant memory regions on GPUs.
- [ ] Documentation. Well yes the only available documentation on the PACXX runtime and the programming model itself is source code.

# Want to contribute? 
Contributions are always welcome. If you want to contribute to PACXX just open a pull request.

# Publications 

Haidl M, Gorlatch S. 2014. ‘[PACXX: Towards a Unified Programming Model for Programming Accelerators using C++14][1].’ Contributed to the The LLVM Compiler Infrastructure in HPC Workshop at Supercomputing '14, New Orleans. doi: 10.1109/LLVM-HPC.2014.9.

Haidl M, Hagedorn B, Gorlatch S. 2016. ‘[Programming GPUs with C++14 and Just-In-Time Compilation][2].’ Contributed to the Advances in Parallel Computing: On the Road to Exascale, ParCo2015, Edinburgh, Schottland. doi: 10.3233/978-1-61499-621-7-247.

Haidl M, Steuwer M, Humernbrum T, Gorlatch S. 2016. ‘[Multi-Stage Programming for GPUs in Modern C++ using PACXX][3].’ Contributed to the The 9th Annual Workshop on General Purpose Processing Using Graphics Processing Unit, GPGPU '16, Barcelona, Spain. doi: 10.1145/2884045.2884049.

Haidl M, Gorlatch S. 2017. ‘[High-Level Programming for Many-Cores using C++14 and the STL][4].’ International Journal of Parallel Programming 2017. doi: 10.1007/s10766-017-0497-y.

Haidl M, Steuwer M, Dirks H, Humernbrum T, Gorlatch S. 2017. ‘[Towards Composable GPU Programming: Programming GPUs with Eager Actions and Lazy Views][5].’ In Proceedings of the 8th International Workshop on Programming Models and Applications for Multicores and Manycores, edited by Chen Q, Huang Z, 58-67. New York, NY: ACM. doi: 10.1145/3026937.3026942.

Haidl M, Moll S, Klein L, Sun H, Hack S, Gorlatch S. 2017 '[PACXXv2 + RV: An LLVM-based Portable High-Performance Programming Model][6].' In Proceedings of the 4th of the Fourth Workshop on the LLVM Compiler Infrastructure in HPC at Supercomputing '17, Denver, ACM doi: 10.1145/3148173.3148185

[1]:http://ieeexplore.ieee.org/document/7069296/
[2]:http://ebooks.iospress.nl/publication/42662
[3]:https://dl.acm.org/citation.cfm?doid=2884045.2884049
[4]:https://link.springer.com/article/10.1007%2Fs10766-017-0497-y
[5]:https://dl.acm.org/citation.cfm?doid=3026937.3026942
[6]:https://dl.acm.org/citation.cfm?id=3148185

