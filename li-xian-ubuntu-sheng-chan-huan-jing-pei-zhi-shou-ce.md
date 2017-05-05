# 离线生产环境配置手册

## 一、本地源制作

1. 使用14.04.5 ubuntu 安装64bit 服务器版本
2. 使用apt-get update 更新源list ， 使用cat /etc/issue 确认Kernal 版本为4.0

> `root@ubuntu:/etc/apt/sources.list.d# cat /etc/issue`
>
> `Ubuntu 14.04.5 LTS \n \`

   3. ![](/assets/import.png)      

  4. ap-get update及install完成所有依赖软件包后，下载的所有软件包位于本机的/var/cache/apt/archives/目录下，使用tar命令打包



| `$ cd /var/cache/apt/$ tar zcvf ~/apt.tar.gz archives/` |
| :--- |


 5. dpkg-scanpackages命令

然后是在无网络环境的情况下，安装好了Ubuntu14.04系统

| `$ tar zxvf apt.tar.gz -C ~   #假定文件解压到/home/bear下,所有的deb包都在/home/bear/archives下面$ mkdir ~/dists/trusty/main/binary-amd64 -p$ mkdir ~/dists/trusty/main/binary-i386 -p#安装dpkg-dev以便使用dpkg-scanpackages命令$ sudo dpkg -i ~/archives/dpkg-dev_* ~/archives/libdpkg-perl* ~/archives/make_* ~/archives/binutils_*$ dpkg-scanpackages archives/ /dev/null | gzip > ~/dists/trusty/main/binary-amd64/Packages.gz -r$ dpkg-scanpackages archives/ /dev/null | gzip > ~/dists/trusty/main/binary-i386/Packages.gz -r` |
| :--- |


| `$ sudo cp /etc/apt/sources.list /etc/apt/sources.list.ori$ sudo vim /etc/apt/sources.list  #将文件中的内容全部删除,然后写成如下格式deb file:///home/bear/ trusty main` |
| :--- |


然后可以测试一下自行搭建的源是否可以正常使用了

| `sudo apt-get updatesudo apt-get install XXX` |
| :--- |




