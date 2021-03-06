EfiFs - EFI File System Drivers
===============================

This is a GPLv3+ implementation of standalone EFI File System drivers, based on
the [GRUB 2.0](http://www.gnu.org/software/grub/) __read-only__ drivers.

For additional info as well as precompiled drivers, see https://efi.akeo.ie

## Requirements

* [Visual Studio 2017](https://www.visualstudio.com/vs/community/) (Windows)
  preferably with Update 4 or later for ARM64 compilation support, MinGW
  (Windows), gcc (Linux) or [EDK2](https://github.com/tianocore/edk2).
* A git client able to initialize/update submodules
* [QEMU](http://www.qemu.org) __v2.7 or later__ if debugging with Visual Studio
  (NB: You can find QEMU Windows binaries [here](https://qemu.weilnetz.de/w64/))

## Compilation

### Common

* Fetch the git submodules with `git submodule init` and `git submodule update`.
  __NOTE__ This only works if you cloned the directory using `git`.

### Visual Studio (non EDK2)

* Apply `0000-GRUB-fixes-for-MSVC.patch` to the `grub\` subdirectory. This
  applies changes that are required for successful MSVC compilation.
* Open the solution file and hit `F5` to compile and debug the default driver.

### gcc (non EDK2)

* Run `make` in the top directory. If needed you can also issue something like
  `make ARCH=<arch> CROSS_COMPILE=<tuple>` where `<arch>` is one of `ia32`,
  `x64`, `arm` or `aa64` (the __official__ UEFI abbreviations for an arch, as
  used in `/efi/boot/boot[ARCH].efi`) and `<tuple>` is the one for your cross-
  compiler, such as `arm-linux-gnueabihf-`.
  e.g. `make ARCH=aa64 CROSS_COMPILE=aarch64-linux-gnu-`

### EDK2

* If using Visual Studio, apply `0000-GRUB-fixes-for-MSVC.patch` to the `grub\`
  subdirectory.
* Open an elevated command prompt and create a symbolic link called `EfiFsPkg`,
  inside your EDK2 directory, to the EfiFs source. On Windows, from an elevated
  prompt, you could run something like `mklink /D EfiFsPkg C:\efifs`, and on
  Linux `ln -s ../efifs EfiFsPkg`.
* From a command prompt, set Grub to target the platform you are compiling for
  by invoking:
  * (Windows) `set_grub_cpu.cmd <arch>`
  * (Linux) `./set_grub_cpu.sh <arch>`  
  Where `<arch>` is one of `ia32`, `x64`, `arm` or `aarch64`.  
  Note that you __MUST__ invoke the `set_grub_cpu` script __every time you
  switch target__.
* After having invoked `Edk2Setup.bat` (Windows) or `edksetup.sh` (Linux) run
  something like:  
  ```
  build -a X64 -b RELEASE -t <toolchain> -p EfiFsPkg/EfiFsPkg.dsc
  ```  
  where `<toolchain>` is something like `VS2015` (Windows) or `GCC5` (Linux).  
  NB: To build an individual driver, such as NTFS, you can also use something
  like:  
  ```
  build -a X64 -b RELEASE -t <toolchain> -p EfiFsPkg/EfiFsPkg.dsc -m EfiFsPkg/EfiFsPkg/Ntfs.inf
  ```
* Note that, provided that you cloned a recent EDK2 from git, you should be able
  to use `VS2017` as your EDK2 toolchain, including for buidling the ARM or 
  ARM64 drivers, with something like:
  ```
  build -a AARCH64 -b RELEASE -t VS2017 -p EfiFsPkg/EfiFsPkg.dsc
  ```
* A Windows script to build all the drivers for all architectures, using EDK2 +
  VS2017 is also provided as `edk2_build_all_drivers.cmd`.

## Testing

If QEMU is installed, the Visual Studio solution will set up and test the
drivers using QEMU (by also downloading a sample image for each target file
system). Note however that VS debugging expects a 64-bit version of QEMU to be
installed in `C:\Program Files\qemu\` (which you can download [here](https://qemu.weilnetz.de/w64/)).
If that is not the case, you should edit `.msvc\debug.vbs` accordingly.

For testing outside of Visual Studio, make sure you have at least one disk with
a target partition using the target filesystem, that is not being handled by
other EFI filesystem drivers.
Then boot into the EFI shell and run the following:
* `load fs0:\<fs_name>_<arch>.efi` or wherever your driver was copied
* `map -r` this should make a new `fs#` available, eg `fs2:`
* You should now be able to navigate and access content (in read-only mode)
* For logging output, set the `FS_LOGGING` shell variable to 1 or more
* To unload use the `drivers` command, then `unload` with the driver ID

## Visual Studio 2017 and ARM/ARM64 support

Please be mindful that, to enable ARM/ARM64 compilation support in Visual
Studio 2017, you __MUST__ go to the _Individual components_ screen in the setup
application and select the ARM compilers and libraries there, as they do __NOT__
appear in the default _Workloads_ screen:

![VS2017 Individual Components](http://files.akeo.ie/pics/VS2017_Individual_Components2.png)

You also need to ensure that you have Windows SDK 10.0.14393.0 or later
installed, as this is the minimum version that provides the required ARM64
static libraries.

## Additional Notes

This is a pure GPLv3+ implementation of EFI drivers. Great care was taken not to
use any code from non GPLv3 compatible sources, such as rEFInd's `fsw_efi`
(GPLv2 only) or Intel's FAT driver (requires an extra copyright notice).

## Bonus: Commands to compile EfiFs using EDK2 on a vanilla Debian GNU/Linux 9.1

As root:
```
apt-get install nasm uuid-dev gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
cd /usr/src
git clone https://github.com/tianocore/edk2.git
git clone https://github.com/pbatard/efifs.git
cd efifs
git submodule init
git submodule update
cd edk2
ln -s ../efifs EfiFsPkg
make -C /usr/src/edk2/BaseTools/Source/C
export GCC5_ARM_PREFIX=arm-linux-gnueabihf-
export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-
source edksetup.sh
./EfiFsPkg/set_grub_cpu.sh X64
build -a X64 -b RELEASE -t GCC5 -p EfiFsPkg/EfiFsPkg.dsc
./EfiFsPkg/set_grub_cpu.sh IA32
build -a IA32 -b RELEASE -t GCC5 -p EfiFsPkg/EfiFsPkg.dsc
./EfiFsPkg/set_grub_cpu.sh ARM
build -a ARM -b RELEASE -t GCC5 -p EfiFsPkg/EfiFsPkg.dsc
./EfiFsPkg/set_grub_cpu.sh AARCH64
build -a AARCH64 -b RELEASE -t GCC5 -p EfiFsPkg/EfiFsPkg.dsc
```
