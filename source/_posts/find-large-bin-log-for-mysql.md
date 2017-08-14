title: 一次mysql运维经验-Binlog过大导致磁盘空间不足
date: 2017-08-14 14:25:27
categories: programming
tags:
- mysql
- linux
- mysql-bin-log
---

# 背景：
---

3天之前安装的一台阿里云测试环境mysql服务器。今天发现磁盘空间居然不够使用了。通过如下命令找到机器上超过50MB的文件。

> * 调整下面的{/path/to/directory/} 为根路径: /
> * 调整下面的{size-in-kb} 为: 50000，单位kb，即50MB。

<!--more-->

```
find {/path/to/directory/} -type f -size +{size-in-kb}k -exec ls -lh {} \; | awk '{ print $9 ": " $5 }'
```

原来是mysql的binlog产生超大的日志文件.

```
-rw-r----- 1 mysql mysql 1086460413 Aug 11 18:48 mybinlog.000029
-rw-r----- 1 mysql mysql 1075039378 Aug 11 20:54 mybinlog.000030
-rw-r----- 1 mysql mysql 1075022833 Aug 11 23:00 mybinlog.000031
...
```

ref: 
* https://www.cyberciti.biz/faq/how-do-i-find-the-largest-filesdirectories-on-a-linuxunixbsd-filesystem/

* https://www.cyberciti.biz/faq/find-large-files-linux/

# 问题原因分析：
---

* 阿里云ECS默认mounted的disk device只有40GB。

```
df -h

# output
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        40G  8.0G   38G  100% /
devtmpfs        7.8G     0  7.8G   0% /dev
tmpfs           7.8G     0  7.8G   0% /dev/shm
tmpfs           7.8G  360K  7.8G   1% /run
tmpfs           7.8G     0  7.8G   0% /sys/fs/cgroup
tmpfs           1.6G     0  1.6G   0% /run/user/0
```

* 阿里云ECS有100GB的disk partition并没有挂载。

```
lsblk

# output
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0     11:0    1 1024M  0 rom
vda    253:0    0   40G  0 disk
└─vda1 253:1    0   40G  0 part /
vdb    253:16   0  100G  0 disk
```

ref: https://askubuntu.com/questions/626353/how-to-list-unmounted-partition-of-a-harddisk-and-mount-them

* my.cnf当中设置的binlog大小为1GB，binlog日志过期时间为7天，造成binlog太大。

```
vim /etc/my.cnf

# output
max_binlog_size = 1G
expire_logs_days = 7
```

# Solution：
---

## 删除掉无用的binlog：

由于这台DB server还没有正式使用，因此我们可以首先删除掉之前所有产生的binlog.

```
PURGE BINARY LOGS TO 'mybinlog.000058';
```

执行上述语句，可以删除掉mybinlog.000058之前所有产生的日志。

## 将unmounted的分区vdb进行挂载：

阿里云有提供auto_fdisk_ssd.sh，用于自动挂载分区。

```
touch auto_fdisk_ssd.sh
vim auto_fdisk_ssd.sh
```

```
#/bin/bash
#########################################
#Function:    auto fdisk
#Usage:       bash auto_fdisk.sh
#Author:      Customer service department
#Company:     Alibaba Cloud Computing
#Version:     4.0
#########################################

count=0
tmp1=/tmp/.tmp1
tmp2=/tmp/.tmp2
>$tmp1
>$tmp2
fstab_file=/etc/fstab

#check lock file ,one time only let the script run one time 
LOCKfile=/tmp/.$(basename $0)
if [ -f "$LOCKfile" ]
then
  echo -e "\033[1;40;31mThe script is already exist,please next time to run this script.\033[0m"
  exit
else
  echo -e "\033[40;32mStep 1.No lock file,begin to create lock file and continue.\033[40;37m"
  touch $LOCKfile
fi

#check user
if [ $(id -u) != "0" ]
then
  echo -e "\033[1;40;31mError: You must be root to run this script, please use root to install this script.\033[0m"
  rm -rf $LOCKfile
  exit 1
fi

#check disk partition
check_disk()
{
  >$LOCKfile
  device_list=$(fdisk -l|grep "Disk"|grep "/dev"|awk '{print $2}'|awk -F: '{print $1}'|grep "vd")
  for i in `echo $device_list`
  do
    device_count=$(fdisk -l $i|grep "$i"|awk '{print $2}'|awk -F: '{print $1}'|wc -l)
    echo 
    if [ $device_count -lt 2 ]
    then
      now_mount=$(df -h)
      if echo $now_mount|grep -w "$i" >/dev/null 2>&1
      then
        echo -e "\033[40;32mThe $i disk is mounted.\033[40;37m"
      else
        echo $i >>$LOCKfile
        echo "You have a free disk,Now will fdisk it and mount it."
      fi
    fi
  done
  disk_list=$(cat $LOCKfile)
  if [ "X$disk_list" == "X" ]
  then
    echo -e "\033[1;40;31mNo free disk need to be fdisk.Exit script.\033[0m"
    rm -rf $LOCKfile
    exit 0
  else
    echo -e "\033[40;32mThis system have free disk :\033[40;37m"
    for i in `echo $disk_list`
    do
      echo "$i"
      count=$((count+1))
    done
  fi
}

#check os
check_os()
{
  os_release=$(grep "Aliyun Linux release" /etc/issue 2>/dev/null)
  os_release_2=$(grep "Aliyun Linux release" /etc/aliyun-release 2>/dev/null)
  if [ "$os_release" ] && [ "$os_release_2" ]
  then
    if echo "$os_release"|grep "release 5" >/dev/null 2>&1
    then
      os_release=aliyun5
      modify_env
     fi
  fi
}

#install ext4
modify_env()
{
  modprobe ext4
  yum install e4fsprogs -y
}

#fdisk ,formating and create the file system
fdisk_fun()
{
fdisk -S 56 $1 << EOF
n
p
1


wq
EOF

sleep 5
mkfs.ext4 ${1}1
}

#make directory
make_dir()
{
  echo -e "\033[40;32mStep 4.Begin to make directory\033[40;37m"
  now_dir_count=$(ls /|grep "alidata*"|awk -F "data" '{print $2}'|sort -n|tail -1)
  if [ "X$now_dir_count" ==  "X" ]
  then
    for j in `seq $count`
    do
      echo "/alidata$j" >>$tmp1
      mkdir /alidata$j
    done
  else
    for j in `seq $count`
    do
      k=$((now_dir_count+j))
      echo "/alidata$k" >>$tmp1
      mkdir /alidata$k
    done
  fi
 }

#config /etc/fstab and mount device
main()
{
  for i in `echo $disk_list`
  do
    echo -e "\033[40;32mStep 3.Begin to fdisk free disk.\033[40;37m"
    fdisk_fun $i
    echo "${i}1" >>$tmp2
  done
  make_dir
  >$LOCKfile
  paste $tmp2 $tmp1 >$LOCKfile
  echo -e "\033[40;32mStep 5.Begin to write configuration to /etc/fstab and mount device.\033[40;37m"
  while read a b
  do
    if grep -v ^# $fstab_file|grep ${a} >/dev/null
    then
      sed -i "s=${a}*=#&=" $fstab_file
    fi
    echo "${a}             $b                 ext4    defaults        0 0" >>$fstab_file
  done <$LOCKfile
  mount -a
}

#=========start script===========
echo -e "\033[40;32mStep 2.Begin to check free disk.\033[40;37m"
check_os
check_disk
main
df -h
rm -rf $LOCKfile $tmp1 $tmp2
```

执行完成了之后，相关设置已经被mounted。

```
lsblk

# output

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0     11:0    1 1024M  0 rom
vda    253:0    0   40G  0 disk
└─vda1 253:1    0   40G  0 part /
vdb    253:16   0  100G  0 disk
└─vdb1 253:17   0  100G  0 part /alidata1
```

## 迁移binlog至新设置的分区/alidata1下：

```
# create mysql binlog directory

mkdir -p /alidata1/mysql/3306/data/mybinlog

# 由于使用root账号登录，而mysql是用mysql用户/用户组运行的。
# 因此需要调整目录owner。
chown -R mysql:mysql /alidata1/mysql/
```

vim /etc/my.cnf 修改之前的mysql配置。

```
slow_query_log_file = /alidata1/mysql/3306/data/slow.log
log-error = /alidata1/mysql/3306/error.log
log-bin = /alidata1/mysql/3306/data/mybinlog

max_binlog_size = 100M
expire_logs_days = 3
```

ref:
* http://jjdai.zhupiter.com/2014/07/%E5%A6%82%E4%BD%95%E7%A7%BB%E9%99%A4%E4%B8%A6%E5%8F%96%E6%B6%88mysql%E7%9A%84binary-log%E6%AA%94%E6%A1%88/

* https://www.cyberciti.biz/faq/what-is-mysql-binary-log/

