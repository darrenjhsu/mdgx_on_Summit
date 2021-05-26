# mdgx_on_Summit
Note to compile `mdgx` on Summit with CUDA enabled

I needed to use `mdgx` on Summit to simulate a lot of small systems using its CUDA feature. Turns out it was a bit hard to compile the code. Below is my note on how I got it to work - not tested though.

The starting point was from this page: http://dev-archive.ambermd.org/202006/0000.html . The `summit_install.sh` script contains some valuable information.

I was building the `mdgx` from `AmberTools` from **Amber20**.


## Setting up mdgx on Summit

First, install `miniconda` (kudos to https://github.com/BSDExabio/OpenMM-on-Summit)

```bash
module load cuda/10.1.105 gcc/8.1.1
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-ppc64le.sh
bash Miniconda3-latest-Linux-ppc64le.sh -b -p miniconda
# Initialize your ~/.bash_profile
miniconda/bin/conda init bash
source ~/.bashrc
```

Create an environment and install some packages for amber

```bash
conda create -n amber python==3.7
conda activate amber
conda install numpy scipy matplotlib
```

`module load` things

```bash
module load gcc/6.4.0 spectrum-mpi cuda \
  cmake readline zlib bzip2 boost \
  netcdf netcdf-cxx4 netcdf-fortran parallel-netcdf \
  openblas netlib-lapack fftw
```

The following procedure assumes you have the source code for AmberTools (usually in a folder called `amber20_src` when you untar the file), and you want to install it to somewhere like `/ccs/home/<userID>/software/amber`. We will use `ccmake` from `cmake`. `ccmake` is the visual CMake configuration tool.

```bash
cd /path/to/amber20_src
mkdir build
cd build
ccmake ..
```

Right now the cache will be empty (if the `build` directory is empty to start with):

```bash
EMPTY CACHE
```

Press `c` to configure. Almost immediately the configure will fail, saying `Configuring incomplete, errors occurred!`. Press `e` to exit screen. You will see a few more options pop up:

```
BUILD_HOST_TOOLS                *OFF
CMAKE_INSTALL_POSTFIX           *
CMAKE_INSTALL_PREFIX            */usr/local
COLOR_CMAKE_MESSAGES            *ON
COMPILER                        *
HOST_TOOLS_DIR                  *
USE_HOST_TOOLS                  *OFF
```

Make the following changes:

```bash
CMAKE_INSTALL_PREFIX    /ccs/home/<userID>/software/amber
COMPILER                GNU
```

Configure again, this time it complains about the Python environment. Press `e` and see a lot more options. Make the following changes:

```bash
CUDA                    ON
DOWNLOAD_MINICONDA      OFF
OPENMP                  ON
```

Configure again. This time it takes longer and starts checking various header files. When it's done without errors, return to the options, press `t` to toggle advanced mode (there will be many more options), and make the following changes (use `/` to search for an option):

```bash
DISABLE_TOOLS           addles;amberlite;ambpdb;antechamber;cifparse;cphstats;cpptraj;emil;etc;gbnsr6;gem.pmemd;leap;mm_pbsa;mmpbsa_py;moft;nab;ndiff-2.00;nfe-umbrella-slice;nmode;nmr_aux;packmol_memgen;paramfit;parmed;pbsa;pdb4amber;pymsmt;pysander;pytraj;reduce;rism;sander;saxs;sebomd;sff;sqm;xray;xtalutil
```

As the option reads, this disables all other tools being built, saving considerable amount of time. Optionally, turn on `CMAKE_VERBOSE_MAKEFILE` also visible in the advanced mode if you find reading the compilation commands useful. Configure again. This time in the keys section, there is an additional option `[g] Generate`. (If not, configure again.) Press `g` to generate. The program should exit itself.

Before doing `make`, we need to add an additional line to the `CMakeLists.txt` in the `AmberTools/src/mdgx` folder:

```bash
cd ..
cd AmberTools/src/mdgx
vim CMakeLists.txt
```

Inside the file, find this line:
```cmake
cuda_add_executable(mdgx.cuda ${MDGX_SOURCES} ${MDGX_CUDA_SOURCES} OPTIONS -DCUDA)
```

Below it, add this line. It forces the C and C++ compilers to take the `-DCUDA` flag, which is important in enabling the CUDA functionalities.

```cmake
target_compile_options(mdgx.cuda PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-DCUDA> $<$<COMPILE_LANGUAGE:C>:-DCUDA>)
```

Now return to the `build` folder and build

```bash
cd ../../../build
make -j 8 | tee build.log
make install | tee -a build.log
```

## Testing

With `mdgx` set up, let's test its speed and whether we actually got the CUDA to work.

In the `tests` folder, go to `CPU` and run

```bash
/ccs/home/<userID>/software/amber/bin/mdgx -O -i mdgxCPU.in
```

Wait for a while and look at the output file `Pro1AIE_MTS.out` which prints timing information. A copy of my benchmarking is in the `ExampleOutput` folder. To run 5000 steps on a CPU, it took 25.56 seconds.

Then let's test the GPU version. Go to `GPU` and run

```bash
/ccs/home/<userID>/software/amber/bin/mdgx.cuda -O -i mdgxGPU.in
```

This time to run 50000 steps on GPU, it took about 6.32 seconds. This is a 40x difference with 1 copy. On a GPU `mdgx.cuda` can run multiple copies simultaneously (up to 80 with a V100 GPU and this particular system). It took 9.23 seconds to run 80 copies, 50000 steps each. This is a throughput difference of 2215x.

