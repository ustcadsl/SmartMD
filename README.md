# SmartMD
SmartMD: A High Performance Deduplication  Engine with Mixed Pages. Our paper has been published in ATC'17.

### 0. Environment
Ubuntu 16.04 + KVM

### 1. SmartMD installation

* Download and unzip the 3.14.69 linux kernel source code. （下载并解压3.14.69内核源码）

  ```bash
  cd ~
  # Download
  wget https://cdn.kernel.org/pub/linux/kernel/v3.x/linux-3.14.69.tar.xz
  # Unzip
  tar -xvf linux-3.14.69.tar.xz
  ```

* Change the code of the memory deduplication module to the code of SmartMD. （更换内存重删模块代码为SmartMD的代码）

  ```bash
  git clone https://github.com/ustcadsl/SmartMD.git
  cd ~/linux-3.14.69/mm
  mv ksm.c ksm-ori.c 
  mv migrate.c migrate-ori.c
  cd ../include/linux/ && mv migrate.h migrate-ori.h
  
  cd ~/linux-3.14.69/mm
  cp ~/SmartMD/src/ksm.c ksm.c
  cp ~/SmartMD/src/migrate.c migrate.c
  cp ~/SmartMD/src/migrate.h ~/linux-3.14.69/include/linux/migrate.h
  cp ~/SmartMD/src/.config  ~/linux-3.14.69/
  ```

* Compile your kernel. （编译内核）

  ```bash
  cd ~/linux-3.14.69/ 
  sudo make -j`getconf _NPROCESSORS_ONLN` && sudo make modules_install -j`getconf _NPROCESSORS_ONLN` && sudo make install
  ```

* Modify the booted kernel version to 3.14.69. （修改启动的内核版本为3.14.69）

  ```bash
  # Method 1: install grub-customizer
  sudo add-apt-repository ppa:danielrichter2007/grub-customizer
  sudo apt-get update
  sudo apt-get install grub-customizer
  grub-customizer # This step will pop up a graphical interface, you can choose the kernel version to start first. （这一步会弹出图形界面，可以选择优先启动的内核版本）
  
  # Method 2: update /etc/default/grub
  # Check whether the newly compiled kernel is successfully installed
  grep -A100 submenu  /boot/grub/grub.cfg |grep "Ubuntu, with Linux 3.14.69"
  # The correct results is "Ubuntu, with Linux 3.14.69". Next, update /etc/default/grub with following contents. 如果结果中出现Ubuntu, with Linux 3.14.69则说明内核安装成功，接下来设置启动内核为3.14.69，只需要将内容修改为下面#<<<之间的内容即可。
  #<<<
  GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 3.14.69"
  #GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 4.4.0"
  #GRUB_HIDDEN_TIMEOUT="5"
  GRUB_HIDDEN_TIMEOUT_QUIET="true"
  GRUB_TIMEOUT="10"
  GRUB_DISTRIBUTOR="`lsb_release -i -s 2> /dev/null || echo Debian`"
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
  GRUB_CMDLINE_LINUX=""
  #<<<
  # Enable configuration. （最后继续执行如下指令以启用配置）
  sudo update-grub
  ```

* Use `sudo reboot` to reboot your machine （执行`sudo reboot`指令重启机器）

* After the machine restarts, use `uname -a` to check whether the current kernel version is 3.14.69. （机器重启后，使用`uname -a`来查看当前的版本是否为3.14.69）

### 2. Benchmark installation

**For benchmark information, please see[our paper](https://www.usenix.org/conference/atc17/technical-sessions/presentation/guo-fan)**。

**The Benchmark installation process is as follows. （Benchmark安装过程如下）**

* Copy benchmark.zip to the virtual machine through the scp command. （将benchmark.zip通过scp拷贝到虚拟机中）

* Enter your virtual machine and use `unzip -d benchmark.zip` to unzip benchmark.zip. （进入到虚拟机中并使用`unzip -d benchmark.zip`解压test.tar.gz）

* Run benchmarks. （测试Benchmark能否正常运行，不能的话需要安装相应依赖）

  ```bash
  # 1. Graph500
  cd ~/graph500-master && make
  ./seq-csr/seq-csr -s 22 -e 16 # The main indicator of this application is harmonic_mean_TEPS, the bigger the better. （此应用的主要关注的指标为harmonic_mean_TEPS，越大越好）
  
  # 2. Liblinear-2.1
  cd ~/liblinear-2.1 && wget https://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary/url_combined.bz2
  bzip2 -d url_combined.bz2
  time ./train -s 7 url_combined # The main indicator of this application is the execution time, the shorter the better. （此应用的主要关注指标为运行时长，越短越好）
  
  # 3. SPECjbb2005-master
  cd ~/SPECjbb2005-master 
  # Modify the SPECjbb.props file to configure according to personal needs, or you can directly use what we have configured. （首先根据个人需求修改SPECjbb.props文件进行配置，也可以直接用我们已经配置好的）
  # Run benchmark
  chmod +x run.sh && ./run.sh
  
  #4. Sysbench-0.4.8
  sudo apt-get install mysql-server
  # update mysql configuration, just `vim /etc/mysql/mysql.conf.d/mysqld.cnf` or `vim /etc/mysql/my.cnf` （更新mysql的配置文件/etc/mysql/mysql.conf.d/mysqld.cnf或/etc/mysql/my.cnf）
  sudo service mysql restart
  sudo apt-get install sysbench
  # Enter mysql （进入mysql中）
  mysql -u root -p
  # Create database. （创建数据库）
  create database sbtest;
  exit #Exit the mysql database. （退出mysql数据库）
  # Create test data, remember to modify the mysql password in the following command. （创建测试数据，记得修改下面指令中mysql的密码）
  sysbench --test=oltp --oltp-test-mode=nontrx --mysql-table-engine=innodb --
  mysql-user=root --db-driver=mysql --num-threads=8 --max-requests=5000000 --
  oltp-nontrx-mode=select --mysql-db=sbtest --oltp-table-size=7000000 --oltptable-name=sbtest --mysql-host=127.0.0.1 --mysqlsocket=/var/run/mysqld/mysqld.sock --mysql-password=123 prepare
  # Test, also remember to modify the mysql password in the following command. （进行测试，同样记得修改下面指令中mysql的密码）
  time sysbench --test=oltp --oltp-test-mode=nontrx --mysql-table-engine=innodb --
  mysql-user=root --db-driver=mysql --num-threads=8 --max-requests=5000000 --
  oltp-nontrx-mode=select --mysql-db=sbtest --oltp-table-size=7000000 --oltptable-name=sbtest --mysql-host=127.0.0.1 --mysqlsocket=/var/run/mysqld/mysqld.sock --mysql-password=123 run
  # Note: The performance indicator is the number of transactions processed per second. （注：性能指标为每秒处理的事务数）
  # If you need to read the test data into the memory in advance, you can use the following command.  （如果需要提前将测试数据读入内存，可以使用如下指令）
  mysql -u root -p 
  use sbtest; 
  select count(id) from (select * from sbtest) aa;
  # If you need to recreate the test data, just delete the original data first. （如果需要重新创建测试数据，需要先删除原先的数据）
  drop table sbtest;
  # To view the cache hit situation, you can use the following commands. （查看cache hit情况可以使用如下指令）
  show global status like 'innodb%read%';
  exit 
  ```

### 3. SmartMD evaluation
**<font color='red'>SmartMD needs the support of transparent huge pages, you can execute the command `sudo bash -c "echo always> /sys/kernel/mm/transparent_hugepage/enabled"` to enable it. In addition, you need to disable KSM when the machine starts, and this can be achieved by executing a startup script.</font>**

**<font color='red'>使用SmartMD需要开启透明大页，可以执行指令：`sudo bash -c "echo always > /sys/kernel/mm/transparent_hugepage/enabled"`来开启。此外，还需要关闭KSM的开机运行，可以执行开机脚本来关闭KSM。</font>**

Virtual machine CPU pinning configuration: （虚拟机CPU绑定配置）

```bash
# 1. Update /etc/libvirt/qemu/xxx.xml, where xxx represents the name of the virtual machine. For example, if you want to bind the vcpu of vm01 to physical core 0, just modify the 13th content of vm01.xml as follows, and the other content does not need to be changed. （可以通过修改/etc/libvirt/qemu/xxx.xml文件完成，其中xxx表示虚拟机的名称。比如将vm01的vcpu绑定到物理core 0上，可以第13内容如下，而其他内容无需改动）

<vcpu placement='static' cpuset='0'>1</vcpu> 

# 2. Enable configuration. （应用修改好的xml配置）
virsh define vm01.xml
```

Configure SmartMD. （SmartMD运行过程中的配置）

```bash
echo 1024 >/sys/kernel/mm/ksm/pages_to_scan
echo 20 >/sys/kernel/mm/ksm/sleep_millisecs
echo 2600 >/sys/kernel/mm/ksm/merge_sleep_millisecs # That is, check_interval in the paper. （即论文中的check_interval）
```

Experiments. （开始跑实验进行测试）

```bash
# 1. Ssh to 4 VMs, and then run the benchmarks in the 4 VMs at the same time. Here is graph500 as an example. You can use a script similar to the following.  （使用ssh连接4个VM，然后同时运行4个VM中的benchmark，注意每一次测试中各个VM中只运行1个benchmark且各个VM运行相同的benchmark，此处以graph500为例，可以使用类似下面的脚本）
for i in `seq 1 4`
do
	ssh vm0$i cd ~/graph500-master && ./seq-csr/seq-csr -s 22 -e 16 > graph.out
done
# 2. Enable KSM. （运行KSM）
sudo bash -c "echo 1 >/sys/kernel/mm/ksm/run"
# 3. Collect all results. （等待所有的benchmark运行结束后收集结果）
```

**<font color='red'>More design detail, please see our [paper](https://www.usenix.org/conference/atc17/technical-sessions/presentation/guo-fan)</font>**。
