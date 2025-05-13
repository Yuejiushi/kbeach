##                                   centos安装jansson、valgrind 

#### 一、内存检测软件：valgrind 

```shell
wget https://sourceware.org/pub/valgrind/valgrind-3.16.1.tar.bz2(本机下载加xshell的rz)
tar -xjvf valgrind-3.16.1.tar.bz2 (或许需要用到yum install bzip2)
cd valgrind-3.16.1/
yum -y install automake
{
	http://ftp.gnu.org/gnu/autoconf
	tar -zxvf autoconf-2.69.tar.gz 
	cd autoconf-2.69
	./configure
	make;make install
	autoconf --version
}
./autogen.sh
./configure
make
sudo make install
```

<img src="C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20200818160019976.png" alt="image-20200818160019976" style="zoom: 67%;" />

命令指示：valgrind --tool=memcheck --leak-check=full --trace-children=yes --track-fds=yes --show-reachable=yes   ./xxxx

测试：valgrind --tool=memcheck --leak-check=full --trace-children=yes --track-fds=yes --show-reachable=yes  --log-file=./valgrind_report.log  ./bin/mysql-sniffer -i lo -p 1433

```less
https://www.cnblogs.com/gmpy/p/14778243.html

valgrind 将内存泄漏分为 4 类。

明确泄漏（definitely lost）：内存还没释放，但已经没有指针指向内存，内存已经不可访问
间接泄漏（indirectly lost）：泄漏的内存指针保存在明确泄漏的内存中，随着明确泄漏的内存不可访问，导致间接泄漏的内存也不可访问
可能泄漏（possibly lost）：指针并不指向内存头地址，而是指向内存内部的位置
仍可访达（still reachable）：指针一直存在且指向内存头部，直至程序退出时内存还没释放。
```



#### 二、安装jansson

Jshon的安装需要Jansson支持：[jansson](http://www.digip.org/jansson/)

```less
wget http://www.digip.org/jansson/releases/jansson-2.5.tar.gz
tar -zxvf jansson-2.5.tar.gz
cd jansson-2.5
./configure  && make && make install
cd /root/soft
wget http://kmkeen.com/jshon/jshon.tar.gz
tar -zxvf  jshon.tar.gz
cd jshon-20120914
make

```

测试

```less
echo '{"40154":"SND-VN-709", "40163":"SND-VN-710"}' | ./jshon
```

![image-20201111154751429](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201111154751429.png)

在这里可能会出现问题
error while loading shared libraries: libjansson.so.4: cannot open shared object file: No such file or directory
解决问题

```less
ls /usr/local/lib/
libjansson.a  libjansson.la  libjansson.so  libjansson.so.4  libjansson.so.4.7.0  pkgconfig

ln -s /usr/local/lib/libjansson.so.4 /usr/lib/libjansson.so.4
ldconfig
```

