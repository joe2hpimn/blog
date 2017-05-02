# 编译Python 2.7指南

## 1、安装编译环境

yum groupinstall "Development tools"    
可直接将GCC升级到4.8或以上版本，如果已安装高版本的GCC可以略过该步骤。

## 2、安装依赖的包
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel   

## 3、下载Python
wget https://www.python.org/ftp/python/2.7.11/Python-2.7.11.tgz   
tar zxvf Python-2.7.11.tgz  
cd Python-2.7.11  

## 4、编译安装
./configure --prefix=/data/pipelinedb/python --enable-shared  
make && make install     
注：必须指定`--enable-shared`，否则在编译Pipelinedb时会出错。

## 5、直接调用
/usr/local/python/bin/python  

## 6、替换旧版本
ln -sf /usr/local/python/bin/python /usr/bin/python   

## 7、查看版本
python -V    
Python 2.7.11   
