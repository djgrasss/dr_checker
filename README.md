# DR.CHECKER : A Soundy Analysis for Linux Kernel Drivers
This repo contains all the sources, including setup scripts.
### Tested on
Ubuntu >= 14.04.5 LTS

## Setup
Our implementation is based on LLVM, specifically LLVM 3.8. We also need tools like `c2xml` to parse headers.
We have created a single script, which downloads and builds all the required tools.
```
cd helper_scripts
python setup_drchecker.py --help
usage: setup_drchecker.py [-h] [-b TARGET_BRANCH] [-o OUTPUT_FOLDER]

optional arguments:
  -h, --help        show this help message and exit
  -b TARGET_BRANCH  Branch (i.e version) of the LLVM to setup. Default:
                    release_38 e.g., release_38
  -o OUTPUT_FOLDER  Folder where everything needs to be setup.

```
Example:
```
python setup_drchecker.py -o drchecker_deps
```
To complete the setup you also need modifications to your local `PATH` environment variable. The setup script will give you exact changes you need to do.
## Building
This depends on the successful completion of [Setup](#markdown-header-setup).
We have a single script that builds everything, you are welcome.
```
cd llvm_analysis
./build.sh
```
## Running
This depends on the successful completion of [Build](#markdown-header-building).
To run DR.CHECKER on kernel drivers, we need to first convert them into llvm bitcode.
### Building kernel
First, we need to have a buildable kernel. Which means we should be able to compile the kernel using regular build setup. i.e., make.
We first capture the output of `make` command, from this output we extract the extract compilation command.
#### Generating output of `make` (or `makeout.txt`)
Just pass `V=1` and redirect the output to the file.
Example:
```
make V=1 O=out ARCH=arm64 > makeout.txt 2>&1
```
### Running DR.CHECKER analysis
There are several steps to run DR.CHECKER analysis, all these steps are wrapped in a single script `helper_scripts/runner_scripts/run_all.py`
How to run:
```
python run_all.py --help
usage: run_all.py [-h] [-l LLVM_BC_OUT] [-a CHIPSET_NUM] [-m MAKEOUT] [-g COMPILER_NAME] [-n ARCH_NUM] [-o OUT] [-k KERNEL_SRC_DIR] [-skb] [-skl] [-skp] [-ske] [-ski] [-f SOUNDY_ANALYSIS_OUT]

optional arguments:
  -h, --help            show this help message and exit
  -l LLVM_BC_OUT        Destination directory where all the generated bitcode files should be stored.
  -a CHIPSET_NUM        Chipset number. Valid chipset numbers are:
                        1(mediatek)|2(qualcomm)|3(huawei)|4(samsung)
  -m MAKEOUT            Path to the makeout.txt file.
  -g COMPILER_NAME      Name of the compiler used in the makeout.txt, This is
                        needed to filter out compilation commands. Ex: aarch64-linux-android-gcc
  -n ARCH_NUM           Destination architecture, 32 bit (1) or 64 bit (2).
  -o OUT                Path to the out folder. This is the folder, which
                        could be used as output directory during compiling
                        some kernels. (Note: Not all kernels needs a separate out folder)
  -k KERNEL_SRC_DIR     Base directory of the kernel sources.
  -skb                  Skip LLVM Build (default: not skipped).
  -skl                  Skip Dr Linker (default: not skipped).
  -skp                  Skip Parsing Headers (default: not skipped).
  -ske                  Skip Entry point identification (default: not
                        skipped).
  -ski                  Skip Soundy Analysis (default: not skipped).
  -f SOUNDY_ANALYSIS_OUT    Path to the output folder where the soundy analysis output should be stored.

```
#### Example:
We have uploaded a mediatek kernel [33.2.A.3.123.tar.bz2](https://drive.google.com/open?id=0B4XwT5D6qkNmLXdNTk93MjU3SWM). 
First download and extract the above file.

Lets say you extracted the above file in a folder called: `~/mediatek_kernel`

##### Building
```
cd ~/mediatek_kernel
source ./env.sh
cd kernel-3.18
# the following step may not be needed depending on the kernel
mkdir out
make O=out ARCH=arm64 tubads_defconfig
# this following command copies all the compilation commands to makeout.txt
make V=1 -j8 O=out ARCH=arm64 > makeout.txt 2>&1
```
##### Running DR.CHECKER
```
cd <repo_path>/helper_scripts/runner_scripts

python run_all.py -l ~/mediatek_kernel/llvm_bitcode_out -a 1 -m ~/mediatek_kernel/kernel-3.18/makeout.txt -g aarch64-linux-android-gcc -n 2 -o ~/mediatek_kernel/kernel-3.18/out -k ~/mediatek_kernel/kernel-3.18 -f ~/mediatek_kernel/dr_checker_out
```
The above command takes quite some time (30 min - 1hr), all the analysis results will be in the folder: `~/mediatek_kernel/dr_checker_out`, for each entry point a `.json` file will be created which contains all the warnings in JSON format.

#### Things to note:
##### Value for option `-g`
To provide value for option `-g` you need to know the name of the `-gcc` binary used to compile the kernel.
An easy way to know this would be to `grep` for `gcc` in `makeout.txt` and you will see compiler commands from which you can know the '-gcc` binary name.

For our example above, if your `grep gcc makeout.txt`, you will see lot of lines like below:
```
aarch64-linux-android-gcc -Wp,-MD,fs/jbd2/.transaction.o.d  -nostdinc -isystem ...
```
So, the value for `-g` should be `aarch64-linux-android-gcc`. 

If the kernel to be built is 32-bit then the binary most likely will be `arm-eabi-gcc`

##### Value for option `-a`
Depeding on the chipset type, you need to provide corresponding number.

##### Value for option `-o`
This is the path of the folder provided to the option `O=` for `make` command, while building kernel.

Not all kernels need a separate out path. You may build kernel by not providing an option `O`, in which case you SHOULD NOT provide value for that option while running `run_all.py`.

Have fun!!

## Contact
Aravind Machiry (machiry@cs.ucsb.edu)
