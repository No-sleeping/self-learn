## OpenSSL curl

```sh
wget https://curl.haxx.se/download/curl-7.55.1.tar.gz
tar -xzvf  curl-7.55.1.tar.gz
cd curl-7.55.1
./configure
make
sudo make install
```

参考：https://blog.csdn.net/qq_37253168/article/details/120201748

## NSS curl

## GnuTLS curl

- 安装nettle-devel  
  
  Nettle是一个加密库，旨在轻松适应任何环境

Nettle是面向对象语言（C++、Python、Pike等）的加密工具包
Nettle用于加密LSH或GNUPG等程序，甚至内核空间

  详细安装步骤见：https://www.cmdschool.org/archives/6670

  软件介绍：http://www.lysator.liu.se/~nisse/nettle/

```sh
# 卸载旧版本的软件包
yum remove nettle*
# 安装编译工具
yum -y install gcc gcc-c++ make expat-devel
# 下载软件包
cd ~
wget https://ftp.gnu.org/gnu/nettle/nettle-3.4.1.tar.gz
# 解压软件包
tar -xf nettle-3.4.1.tar.gz
# 编译安装
./configure --bindir=/usr/bin/ \
            --sbindir=/usr/sbin/ \
            --libexecdir=/usr/libexec/ \
            --sysconfdir=/etc/ \
            --libdir=/usr/lib64/ \
            --includedir=/usr/include/ \
            --datarootdir=/usr/share/ \
            --infodir=/usr/share/info/ \
            --localedir=/usr/share/locale/ \
            --mandir=/usr/share/man/ \
            --docdir=/usr/share/doc/nettle/ \
            --enable-mini-gmp
make
make install
```

- 安装GnuTLS  
  
  GnuTLS是一个安全的通讯库

GnuTLS使用TLS/SSL（传输层安全协议又称安全套接字层）和DTLS协议实现安全通讯
GnuTLS提供一个简单的C语言应用程序编程接口
GnuTLS编程接口用于访问安全通讯协议
GnuTLS编程接口用于解析和编写X.509、PKCS#12和其他需求的API构造

  安装步骤见：https://www.cmdschool.org/archives/6646

```sh
# 下载软件包
cd ~
wget https://www.gnupg.org/ftp/gcrypt/gnutls/v3.6/gnutls-3.6.9.tar.xz
# 解压软件包
tar -xf gnutls-3.6.9.tar.xz
# 预编译软件库
./configure --prefix=/usr/local/gnutls-3.6.9 \
            --enable-static \
            --disable-guile \
            --with-default-trust-store-pkcs11="pkcs11:"
# 成功回显：
...
configure: System files:

  Trust store pkcs11:   pkcs11:
  Trust store dir:
  Trust store file:
  Blacklist file:
  CRL file:
  Configuration file:   /etc/gnutls/config
  DNSSEC root key file: /var/lib/unbound/root.key
...

make
make install 

```

   参考：https://wzxaini9.cn/article/469.html （Linux[CentOS 9]GNUTLS安装）
