# SmartMD
SmartMD: A High Performance Deduplication  Engine with Mixed Pages. Our paper has been published in ATC'17.


### 1. SmartMD installation

* 下载并解压3.14.69内核源码。

  ```bash
  cd ~
  # 下载源码
  wget https://cdn.kernel.org/pub/linux/kernel/v3.x/linux-3.14.69.tar.xz
  # 解压
  tar -xvf linux-3.14.69.tar.xz
  ```

* 更换内存重删模块代码为SmartMD的代码。

  ```bash
  cd linux-3.14.69/mm
  mv ksm.c ksm-ori.c 
  mv migrate.c migrate-ori.c
  mv ../include/linux/migrate.h migrate-ori.h
  
  cp ~/smartmd/smartmd.c ksm.c
  cp ~/smartmd/migrate.c migrate.c
  cp ~/smartmd/migrate.h ~/linux-3.14.69/include/linux/migrate.h
  ```

* 编译内核。

  ```bash
  cp smartmd/.config linux-3.14.69/ #这一步也可以使用make menuconfig命令来选择并生成配置，smartmd使用的是默认的内核配置
  # 进入到目录linux-3.14.69中
  cd ../
  # 编译内核
  sudo make -j`getconf _NPROCESSORS_ONLN` && sudo make modules_install -j`getconf _NPROCESSORS_ONLN` && sudo make install
  ```

* 修改启动的内核版本为3.14.69。

  ```bash
  # 方法一：安装grub-customizer，Ubuntu 16可以使用如下指令安装：
  sudo add-apt-repository ppa:danielrichter2007/grub-customizer
  sudo apt-get update
  sudo apt-get install grub-customizer
  grub-customizer # 这一步会弹出图形界面，可以选择优先启动的内核版本
  
  # 方法二：修改/etc/default/grub文件
  # 查询新编译的内核是否成功安装，执行如下指令：
  grep -A100 submenu  /boot/grub/grub.cfg |grep "Ubuntu, with Linux 3.14.69"
  # 如果上述指令执行的结果中出现Ubuntu, with Linux 3.14.69则说明内核安装成功，接下来设置启动内核为3.14.69，只需要将内容修改为图1所示。然后再继续执行如下指令：
  sudo update-grub
  ```

* **重启机器**，执行`sudo reboot`指令。

* 机器重启后，使用`uname -a`来查看当前的版本是否为3.14.69。

### 2. Benchmark安装

**Benchmark信息请参考[论文](https://www.usenix.org/conference/atc17/technical-sessions/presentation/guo-fan)**。

**Benchmark安装过程如下：**

* 将test.tar.gz通过scp拷贝到虚拟机中。

* 进入到虚拟机中并使用`tar -xvf test.tar.gz`解压test.tar.gz。

* 测试Benchmark能否正常运行，不能的话需要安装相应依赖。

  ```bash
  # 1. 测试graph500
  cd ~/graph500-master && make
  ./seq-csr/seq-csr -s 22 -e 16 # 此应用的主要关注的指标为harmonic_mean_TEPS，越大越好。
  
  # 2. 测试liblinear-2.1
  cd ~/liblinear-2.1 && time ./train -s 7 url_combined # 此应用的主要关注指标为运行时长，越短越好。
  
  # 3. 测试SPECjbb2005-master
  cd ~/SPECjbb2005-master 
  # 首先根据个人需求修改SPECjbb.props文件进行配置，也可以直接用我们已经配置好的
  # 其次运行benchmark
  chmod +x run.sh && ./run.sh
  
  #4. 测试sysbench-0.4.8
  sudo apt-get install mysql-server
  #修改mysql配置文件,vim /etc/mysql/mysql.conf.d/mysqld.cnf或者vim /etc/mysql/my.cnf，根据自己需求而定
  sudo service mysql restart
  sudo apt-get install sysbench
  #进入mysql数据库
  mysql -u root -p
  #创建数据库
  create database sbtest;
  exit #退出mysql数据库
  #创建测试数据，记得修改下面指令中mysql的密码
  sysbench --test=oltp --oltp-test-mode=nontrx --mysql-table-engine=innodb --
  mysql-user=root --db-driver=mysql --num-threads=8 --max-requests=5000000 --
  oltp-nontrx-mode=select --mysql-db=sbtest --oltp-table-size=7000000 --oltptable-name=sbtest --mysql-host=127.0.0.1 --mysqlsocket=/var/run/mysqld/mysqld.sock --mysql-password=123 prepare
  #进行测试，同样记得修改下面指令中mysql的密码
  time sysbench --test=oltp --oltp-test-mode=nontrx --mysql-table-engine=innodb --
  mysql-user=root --db-driver=mysql --num-threads=8 --max-requests=5000000 --
  oltp-nontrx-mode=select --mysql-db=sbtest --oltp-table-size=7000000 --oltptable-name=sbtest --mysql-host=127.0.0.1 --mysqlsocket=/var/run/mysqld/mysqld.sock --mysql-password=123 run
  #注：性能指标为每秒处理的事务数
  #如果需要提前将测试数据读入内存，可以使用如下指令
  mysql -u root -p #进入MySQL数据库
  use sbtest; #进入sbtest database
  select count(id) from (select * from sbtest) aa;
  #如果需要重新创建测试数据，需要先删除原先的数据
  drop table sbtest;
  #查看cache hit情况可以使用如下指令
  show global status like 'innodb%read%';
  exit #退出MySQL数据库
  ```

### 3. SmartMD使用与性能测试

**<font color='red'>使用SmartMD需要开启透明大页，可以执行指令：`sudo bash -c "echo always > /sys/kernel/mm/transparent_hugepage/enabled"来开启。此外，还需要关闭KSM的开机运行，可以执行开机脚本来关闭KSM。`</font>**

虚拟机CPU绑定配置：

```bash
# 1. 可以通过修改/etc/libvirt/qemu/xxx.xml文件完成，其中xxx表示虚拟机的名称。比如将vm01的vcpu绑定到物理core 0上，可以第13内容如下，而其他内容无需改动:
<vcpu placement='static' cpuset='0'>1</vcpu> 

# 2. 应用修改好的xml配置
virsh define vm01.xml
```

SmartMD运行过程中的配置：

```bash
echo 1024 >/sys/kernel/mm/ksm/pages_to_scan
echo 20 >/sys/kernel/mm/ksm/sleep_millisecs
echo 2600 >/sys/kernel/mm/ksm/merge_sleep_millisecs # 即论文中的check_interval
```

开始跑实验进行测试：

```bash
# 1. 使用ssh连接4个VM，然后同时运行4个VM中的benchmark，注意每一次测试中各个VM中只运行1个benchmark且各个VM运行相同的benchmark，此处以graph500为例，可以使用类似下面的脚本：
for i in `seq 1 4`
do
	ssh vm0$i cd ~/graph500-master && ./seq-csr/seq-csr -s 22 -e 16 > graph.out
done
# 2.运行KSM
sudo bash -c "echo 1 >/sys/kernel/mm/ksm/run" # 注意：需要关闭KSM开机运行
# 3.等待所有的benchmark运行结束后收集结果
```

**<font color='red'>SmartMD的设计细节请参考[论文](https://www.usenix.org/conference/atc17/technical-sessions/presentation/guo-fan)</font>**。

