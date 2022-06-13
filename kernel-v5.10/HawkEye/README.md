## 1 Installation of HawkEye based on Linux 5.10

* Download and unzip the 5.10 linux kernel source code.

  ```bash
  cd ~
  # Download
  wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.10.tar.xz
  # Unzip
  tar -xvf linux-5.10.tar.xz
  ```

* Change the code of the memory deduplication module to the code of HawkEye.

  ```bash
  git clone https://github.com/ustcadsl/SmartMD.git
  cp SmartMD/kernel-v5.10/HawkEye/Makefile linux-5.10/
  cp SmartMD/kernel-v5.10/HawkEye/.config linux-5.10/
  cp SmartMD/kernel-v5.10/HawkEye/mm/*  linux-5.10/mm
  cp SmartMD/kernel-v5.10/HawkEye/kernel/fork.c linux-5.10/kernel
  cp SmartMD/kernel-v5.10/HawkEye/include/linux/* linux-5.10/include/linux
  cp SmartMD/kernel-v5.10/HawkEye/events/mmflags.h linux-5.10/events
  cp SmartMD/kernel-v5.10/HawkEye/arch/x86/entry/syscalls/syscall_64.tbl linux-5.10/arch/x86/entry/syscalls/
  ```

* Compile HawkEye.

  ```bash
  cd linux-5.10/
  
  sudo make -j`getconf _NPROCESSORS_ONLN` && sudo make modules_install -j`getconf _NPROCESSORS_ONLN` && sudo make install
  ```

* Modify the booted kernel version to Linux 5.10.

  ```bash
  ```bash
    # Method 1: install grub-customizer
    sudo add-apt-repository ppa:danielrichter2007/grub-customizer
    sudo apt-get update
    sudo apt-get install grub-customizer
    grub-customizer # This step will pop up a graphical interface, you can choose the kernel version to start first.
    
    # Method 2: update /etc/default/grub
    # Check whether the newly compiled kernel is successfully installed
  grep -A100 submenu  /boot/grub/grub.cfg |grep "Ubuntu, with Linux 5.10.0-HawkEye"
    # The correct results is "Ubuntu, with Linux 5.10.0-HawkEye". Next, update /etc/default/grub with following contents.
    #<<<
    GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.10.0-HawkEye"
    #GRUB_HIDDEN_TIMEOUT="5"
    GRUB_HIDDEN_TIMEOUT_QUIET="true"
    GRUB_TIMEOUT="10"
    GRUB_DISTRIBUTOR="`lsb_release -i -s 2> /dev/null || echo Debian`"
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
    GRUB_CMDLINE_LINUX=""
    #<<<
    # Enable configuration.
    sudo update-grub
  ```

* Use `sudo reboot` to reboot your machine.

* After the machine restarts, use `uname -a` to check whether the current kernel version is Linux 5.10.0-HawkEye.

## 2 Run HawkEye
* 