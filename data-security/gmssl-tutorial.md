# 国密证书测试手记


## 前言

有个项目需要过密评，需要将应用系统改造为使用国密证书。

经过调研OpenSSL 1.1.1+版本已经支持了SMX系列的算法，可以生成SM2的公私钥对，但是只支持SHA算法签名的证书，不支持使用SM3算法签名的证书；同时也不支持GM系列的套件。

故这里调研大名鼎鼎的GmSSL，看看对国密证书的支持情况。

## 环境准备

**0. 环境**
```
采用一台云主机： 
CentOS Linux release 7.6.1810 (Core) 
```

**1. 安装GmSSL**

```bash
# 从官网下载GmSSL
wget 'https://github.com/guanzhi/GmSSL/archive/refs/heads/master.zip' -O GmSSL-master.zip

# 解压
unzip GmSSL-master.zip

# Config配置
cd GmSSL-master
./config
make # 漫长的编译过程
make install
```

**2. 排查**
直接执行`gmssl`命令会直接报错：（撰写此文档的时间为2021-05-07日，不清楚后续GmSSL官方是否会修复这个问题）
```bash 
$ gmssl version 
gmssl: error while loading shared libraries: libssl.so.1.1: cannot open shared object file: No such file or directory

$ whereis gmssl
gmssl: /usr/local/bin/gmssl

$ ldd /usr/local/bin/gmssl 
	linux-vdso.so.1 =>  (0x00007ffcba7a1000)
	libssl.so.1.1 => not found
	libcrypto.so.1.1 => not found
	libdl.so.2 => /lib64/libdl.so.2 (0x00007fec70aba000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fec7089e000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fec704d0000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fec70cbe000)

# 经检查，在/usr/local/lib64/目录下能够找到 notfound 的两个so库，并与编译出来的库做对比，MD5是一致的。
$ md5sum  /usr/local/lib64/libcrypto.so libcrypto.so
38bc417f9cdb50b26714781890350d8e  /usr/local/lib64/libcrypto.so
38bc417f9cdb50b26714781890350d8e  libcrypto.so

# 打开Makefile，将 LDFLAG= 修改为 "LDFLAGS= -Wl,-rpath=$(LIBRPATH)"
$ vim Makefile

# 删除编译好的gmssl可执行程序，重新执行Makefile，并拷贝到目标路径
$ rm -fr ./apps/gmssl && make && cp -af ./app/gmssl /usr/local/bin/gmssl

# 再次执行gmssl命令，正常
$ gmssl version
GmSSL 2.5.4 - OpenSSL 1.1.0d  19 Jun 2019


```

**3. 签发证书**

根据国密的标准，服务端证书分为签名证书和加密证书，其中签名证书负责对ClientHello和ServerHello中的随机数进行签名，客户端使用服务器的签名证书公钥进行验签；而后客户端使用服务器的加密证书公钥对ClientKeyExchange的48字节长度PreMasterKey进行加密，服务端使用加密证书私钥对该数据进行解密。而后的流程则与标准RSA步骤相同。

```bash

# 初始化
echo 00 > ca.srl

# 生成CA证书
$ gmssl ecparam -genkey -name sm2p256v1 -noout -out ca.key
$ gmssl req -new -key ca.key -out ca.csr -subj "/C=CN/O=ir0cn/CN=ir0cn-CA"
$ gmssl x509 -req -days 3650 -sm3 -extensions v3_ca -extfile <(printf "[v3_ca]\nbasicConstraints = CA:TRUE\n") -in ca.csr -signkey ca.key -out ca.crt

# 生成服务器签名证书，并使用CA签名
$ gmssl ecparam -genkey -name sm2p256v1 -noout -out ir0.cn.sig.key
$ gmssl req -new -key ir0.cn.sig.key -out ir0.cn.sig.csr -subj "/C=CN/O=ir0cn/CN=*.ir0.cn"
$ gmssl x509 -req -days 3650 -sm3 -extfile <(printf "keyUsage=digitalSignature,nonRepudiation\nsubjectAltName=DNS:api.ir0.cn,DNS:console.ir0.cn") -in ir0.cn.sig.csr -CA ca.crt -CAkey ca.key -out ir0.cn.sig.crt

# 生成服务器加密证书，并使用CA签名
$ gmssl ecparam -genkey -name sm2p256v1 -noout -out ir0.cn.enc.key
$ gmssl req -new -key ir0.cn.enc.key -out ir0.cn.enc.csr -subj "/C=CN/O=ir0cn/CN=*.ir0.cn"
$ gmssl x509 -req -days 3650 -sm3 -extfile <(printf "keyUsage= keyEncipherment,dataEncipherment,keyAgreement\nsubjectAltName=DNS:api.ir0.cn,DNS:console.ir0.cn") -in ir0.cn.enc.csr -CA ca.crt -CAkey ca.key -out ir0.cn.enc.crt

```


## Nginx加载国密证书

**方案1**

调研江南天安的Nginx(基于1.16.0修改[传送门](https://github.com/jntass/Nginx_Tassl))，其修改主要是增加了加密证书、签名证书的处理。


**方案2**

这里选择Nginx与GmSSL的[开源库](https://github.com/guanzhi/GmSSL)集成方案进行测试。**注：nginx支持国密证书，无需编译GmSSL，直接指定原始代码路径即可。**


```bash
# 安装依赖包
$ yum install -y pcre-devel zlib-devel           # centos
$ apt install -y libpcre3-dev libghc-zlib-dev    # ubuntu

# 下载Nginx并解压
$ wget https://github.com/nginx/nginx/archive/refs/tags/release-1.18.0.zip -O nginx-1.18.0.zip
$ unzip nginx-1.18.0.zip 
$ cd nginx-release-1.18.0 

# 修改nginx的编译参数参数，主要是--with-openssl=GmSSL的路径
$ ./auto/configure --prefix=/etc/nginx                                 \
      --sbin-path=/usr/sbin/nginx                                      \
      --conf-path=/etc/nginx/nginx.conf                                \
      --error-log-path=/var/log/nginx/error.log                        \
      --http-log-path=/var/log/nginx/access.log                        \
      --pid-path=/var/run/nginx.pid                                    \
      --lock-path=/var/run/nginx.lock                                  \
      --http-client-body-temp-path=/var/cache/nginx/client_temp        \
      --http-proxy-temp-path=/var/cache/nginx/proxy_temp               \
      --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp           \
      --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp               \
      --http-scgi-temp-path=/var/cache/nginx/scgi_temp                 \
      --with-http_ssl_module                                           \
      --with-http_realip_module                                        \
      --with-http_addition_module                                      \
      --with-http_sub_module                                           \
      --with-http_dav_module                                           \
      --with-http_flv_module                                           \
      --with-http_mp4_module                                           \
      --with-http_gunzip_module                                        \
      --with-http_gzip_static_module                                   \
      --with-http_random_index_module                                  \
      --with-http_secure_link_module                                   \
      --with-http_stub_status_module                                   \
      --with-http_auth_request_module                                  \
      --with-threads                                                   \
      --with-stream                                                    \
      --with-stream_ssl_module                                         \
      --with-http_slice_module                                         \
      --with-mail                                                      \
      --with-mail_ssl_module                                           \
      --with-file-aio                                                  \
      --with-http_v2_module                                            \
      --with-openssl=/GmSSL-master

# 编译
$ make && make install

```

**配置nginx并启动**

```bash
# 编辑nginx配置文件
$ vim /etc/nginx.conf
server {
    listen       443 ssl;
    server_name  *.ir0.cn;

    ssl_protocols TLSv1.1 TLSv1.2;

    # RSA证书
    ssl_certificate_key     certs/ir0.cn.rsa.key;
    ssl_certificate         certs/ir0.cn.rsa.crt;
    
    # ECC证书
    #ssl_certificate_key     certs/ir0.cn.rsa.key;
    #ssl_certificate         certs/ir0.cn.rsa.crt;

    # SM2签名证书
    ssl_certificate_key     certs/ir0.cn.sig.key;
    ssl_certificate         certs/ir0.cn.sig.crt;

    # SM2加密证书
    ssl_certificate_key     certs/ir0.cn.enc.key;
    ssl_certificate         certs/ir0.cn.enc.crt;

    ssl_session_cache       shared:SSL:1m;
    ssl_session_timeout     5m;

    # 加密套件
    #ssl_ciphers ECDHE-SM2-WITH-SMS4-GCM-SM3:SM2-WITH-SMS4-SM3:SM2DHE-WITH-SMS4-SM3:ECDHE-SM2-WITH-SMS4-SM3:ECDHE-SM4-SM3:!aNull:!MD5;
    ssl_ciphers ECDHE-SM2-WITH-SMS4-GCM-SM3:SM2-WITH-SMS4-SM3:SM2DHE-WITH-SMS4-SM3:ECDHE-SM2-WITH-SMS4-SM3:ECDHE-SM4-SM3:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
    ssl_prefer_server_ciphers  on; 

    location / { 
        root   html;
        index  index.html index.htm;
    }   
}


# 启动Nginx
$ nginx -t && nginx

# 使用GmSSL检测证书
$ gmssl s_client -connect 127.0.0.1:443
```


## 国密浏览器


**密信浏览器**

密信浏览器是WoSign基于Chromium开发的支持国密算法的浏览器，试用下来比360浏览器要好用，推荐([传送门](https://www.mesign.com/zh-cn/browser/index.html#dow)) 

**SamariumBorwser**

SamariumBorwser是GmSSL团队研发的基于Chromium的国密浏览器，推荐[传送门](https://github.com/guanzhi/SamariumBrowser)

## 制作的容器

**GmSSL**

```bash
$ docker pull ir0cn/gmssl
$ docker run -it ir0cn/gmssl /bin/bash
root@97324dd477d6:~# gmssl
GmSSL>
```


**GmSSL-Nginx**

```bash
$ docker pull ir0cn/nginx:gmssl
$ docker run -itd ir0cn/nginx:gmssl
7be9b2fb75e0b53366803f7bb0783695dd07566fbb9b3143a0b9ba4f76c2ff7d
$ 
```

