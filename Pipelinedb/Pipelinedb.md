# Pipelinedb数据库源码编译安装指南
## 一 安装GCC
为了获得更好的性能，建议安装最新版的GCC编译器。

### 1.下载GCC源码
wget ftp://mirrors.kernel.org/gnu/gcc/gcc-6.3.0/gcc-6.3.0.tar.gz

### 2.解压
tar -zxvf gcc-6.3.0.tar.gz

### 3.下载编译所需依赖项
cd gcc-6.3.0     					//进入解包后的gcc文件夹  
./contrib/download_prerequisites	//下载依赖项   
cd ..                          		//返回上层目录  

### 4.建立编译输出目录
mkdir gcc-build-6.3.0    
注：创建build目录的当前目录需要有4G以上的剩余磁盘空间，否则最终会编译失败。

### 5.生成makefile文件
mkdir -p /opt/gcc6     
cd gcc-build-6.3.0     
../gcc-6.3.0/configure --enable-checking=release --enable-languages=c,c++ --disable-multilib --prefix=/opt/gcc6    
注意：为了不对操作系统的运行造成影响，需要添加参数“--prefix=/opt/gcc6”将gcc安装到自定义目录。

### 6.编译
make -j4   
接下来就是等待了，整个过程大约40分钟左右。     
注：最好不要在编译过程中再去做别的什么事，整个过程CPU都是满载的，编译时当前目录剩余空间要最少4G以上，否则会编译失败。

### 7.安装
编译结束以后，我们就可以执行安装了：   
make install   

### 8.检查版本
/opt/gcc6/bin/gcc -v

## 二 安装 zeromq
### 1 编译安装
从github上下载zeromq的源代码    
http://zeromq.org/intro:get-the-software    
wget https://github.com/zeromq/libzmq/releases/download/v4.2.0/zeromq-4.2.0.tar.gz    
按一下步骤进行编译安装。为了实现更好的可移植性，需要将zeromq编译到pipelinedb的安装目录。    
tar -zxvf zeromq-4.2.0.tar.gz   
./configure --prefix=/opt/pipelinedb/    
make    
make install   

### 2 修改动态库路径
在安装完zeromq后，需要将zeromq的lib目录添加到系统动态库路径，具体的命令如下所示。    
vi /etc/ld.so.conf  
/opt/pipelinedb/lib   
ldconfig  

## 三 更新libcheck
针对RHEL6（centos6）的linux系统在执行make check的过程中，由于版本的兼容性问题，会造成问题。因此需要更新操作系统的check测试框架。

### 1 删除老版本的check
yum remove check

### 2 安装 check
#### 2.1 check测试框架的详细内容可查阅如下资源。
http://check.sourceforge.net/   
https://libcheck.github.io/check/web/install.html#linuxsource  
https://github.com/libcheck/check/releases

#### 2.2 从sourceforge下载最新源代码。
wget http://downloads.sourceforge.net/project/check/check/0.10.0/check-0.10.0.tar.gz?r=&ts=1482216800&use_mirror=ncu

#### 2.3 编译安装check。
tar -zxvf check-0.10.0.tar.gz  
cd check-0.10.0  
./configure  
make  
make install  

## 四 修改源代码

### 1 下载源代码
wget https://github.com/pipelinedb/pipelinedb/archive/0.9.6.tar.gz    
tar -zxvf 0.9.6.tar.gz     
cd pipelinedb-0.9.6

### 2 源码BUG修复
在安装完check后，需要修复源代码中由于check版本引起的bug。具体的修改操作如下。   
vi src/test/unit/test_hll.c   
vi src/test/unit/test_tdigest.c    
vi src/test/unit/test_bloom.c   
vi src/test/unit/test_cmsketch.c   
vi src/test/unit/test_fss.c   
添加 #include "check.h"  

## 五 Makefile修正
由于pipelinedb的Makefile指定了zeromq的静态路径，因此需要修复libzmq.a路径错误。     
libzmq.a的路径修正操作如下。    
`vi src/Makefile.global.in`    
`LIBS := -lpthread /opt/pipelinedb/lib/libzmq.a -lstdc++ $(LIBS)`   
修改libzmp.a到实际的地址。

## 六 修复test_decoding错误
cd contrib/test_decoding    
mv specs test    
cd ../../     

## 七 编译pipelinedb
export C_INCLUDE_PATH=/usr/local/include:/opt/pipelinedb/include:$C_INCLUDE_PATH     
export LIBRARY_PATH=/usr/local/lib:/opt/pipelinedb/lib:$LIBRARY_PATH    
export USE_NAMED_POSIX_SEMAPHORES=1    
LIBS=-lpthread CC="/opt/gcc6/bin/gcc" CFLAGS="-O3 -flto" PYTHON=/data/pipelinedb/python/bin/python ./configure --prefix=/data/pipelinedb --with-python   
make world -j 32     
make install-world    

`--with-python` 是为了支持pl/pythonu，默认centos6（RHEL6）的python是2.6，我们可以自行编译python2.7，并将其打包到Pipelinedb的安装目录。例如`/data/pipelinedb/python`,python2.7的编译过程请查看 [Python2.7编译指南](https://github.com/joe2hpimn/blog/blob/master/Pipelinedb/%E7%BC%96%E8%AF%91Python2.7.md)
