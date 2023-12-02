---
title: openssl常用命令
date: 2023-11-25 10:05:15
tags:
  - openssl
  - 密钥
  - 公钥
  - 私钥
categories:
  - ["编程语言","shell"]
---
### 查看ca有效期

 openssl x509 -in ca.pem -dates

### 生成私钥 

 openssl  genrsa -out  rsa.key 1024

> $ openssl genrsa -h       
> /
> /*u
> usage: genrsa [args] [numbits]
>  -des           生成的私钥采用DES算法加密
>   -des3          生成的私钥采用DES3算法加密 (168 bit key)
>    -seed          encrypt PEM output with cbc seed
>     -aes128, -aes192, -aes256
>     以上几个都是对称加密算法的指定，因为我们长期会把私钥加密，避免明文存放
>      -out file       私钥输出位置
>       -passout arg    输出文件的密码，如果我们指定了对称加密算法，也可以不带此参数，会有命令行提示你输入密码
>       **/
### 生成公钥
        openssl  rsa -pubout -in rsa.key -out pub.key
### 公钥/私钥加密
        openssl  rsautl -encrypt -in plain.txt -inkey rsa.key -out encrypt.txt
### 私钥解密
         openssl rsautl -decrypt -in encrypt.txt -inkey rsa.key -out plain1.txt
### base64解码
         cat testencrypt |base64 -d > testencrypt.base64

### 将普通私钥转成PKCS#8编码
         openssl pkcs8 -topk8 -in rsa_private_key.pem -out pkcs8_rsa_private_key.pem -nocrypt
