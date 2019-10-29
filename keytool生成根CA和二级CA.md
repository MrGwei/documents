# keytool生成根CA和二级CA

[TOC]

## 1. 生成根CA

### 1.1 生成根CA秘钥库

```
keytool -genkeypair -alias root-ca-integration -keyalg RSA -keysize 2048 -storetype PKCS12 -keystore gwei_root_ca.p12 -validity 3650 -dname "CN=GweiRootCA,OU=GweiRoot,O=GweiRootOrganization,L=BJ,ST=BJ,C=CN" -storepass rootca.com.cn
```

| 参数 | 说明 |
| ---- | ---- |
|      |      |
|      |      |
|      |      |



### 1.2 导出根CA公钥证书

```
keytool -export -keystore gwei_root_ca.p12 -alias root-ca-integration -file gwei_root_ca.pub -storepass rootca.com.cn
```



## 2. 生成二级CA

二级CA证书需要由根证书进行签发，首先需要使用keytool生成二级CA的证书，但是此时的证书还是张自签证书，我们需要从中生成一个二级CA的证书请求（包含了公钥），然后通过将证书请求到rootca签发我们的二级证书，然后我们在将rootca签发的二级CA证书导入到导出证书请求的秘钥库中，完成二级CA的生成。下面是生成二级CA证书命令，此时是张自签证书：



### 2.1 生成二级CA秘钥库

```
keytool -genkeypair -alias sub-ca-integration -keyalg RSA -keysize 2048 -storetype PKCS12 -keystore gwei_sub_ca.p12 -validity 3650 -dname "CN=GweiSubCA,OU=GweiSub,O=GweiSubOrganization,L=BJ,ST=BJ,C=CN" -storepass subca.com.cn
```



### 2.2 导出二级CA公钥证书

```
keytool -export -keystore gwei_sub_ca.p12 -alias sub-ca-integration -file gwei_sub_ca.pub -storepass subca.com.cn
```



### 2.3 从二级CA的秘钥库中生成证书请求

```
keytool -certreq -alias sub-ca-integration -file gwei_sub_ca.csr -keystore gwei_sub_ca.p12 -storepass subca.com.cn
```



### 2.4 使用二级证书请求从根CA中签发证书

```
keytool -gencert -infile gwei_subca.csr -outfile rootca_sign_subca_cert.crt -alias root-ca-integration -keystore gwei_root_ca.p12 -storepass rootca.com.cn -validity 3650
```

通过命令可以查看到发布者是GweiRootCA



### 2.5 导入根CA签发的公钥证书到二级根秘钥库中

```
keytool -importcert -file gwei_root_ca.pub -alias root-ca-integration -storetype PKCS12 -keystore gwei_sub_ca.p12 -storepass subca.com.cn
```



### 2.6 导入根CA签发的证书链

导入根CA签发的证书到二级根秘钥库，

```
keytool -importcert -v -alias sub-ca-integration -file rootca_sign_subca_cert.crt -storetype PKCS12 -keystore gwei_sub_ca.p12 -storepass subca.com.cn
```

Q: Exception:"无法从回复中建立链";

A: 命令行提示错误"无法从回复中建立链"，这是因为在更新被签发证书之前，一定要先将签发证书的机构的信任证书导入到密钥库文件，即将密钥库CA根的证书以其相应的别名导入到密钥库gwei_sub_ca.p12中;

```
keytool -importcert -file gwei_root_ca.pub -alias root-ca-integration -storetype PKCS12 -keystore gwei_sub_ca.p12 -storepass subca.com.cn
```



## 3. 使用二级证书签发用户证书

### 3.1 生成用户秘钥库

```
keytool -genkeypair -alias user1 -keyalg RSA -keysize 2048 -storetype PKCS12 -keystore gwei_user1_ca.p12 -validity 3650 -dname "CN=GweiUser1CA,OU=GweiUser1,O=GweiUser1Organization,L=BJ,ST=BJ,C=CN" -storepass user1ca.com.cn
```



### 3.2 导出用户公钥证书

```
 keytool -export -keystore gwei_user1_ca.p12 -alias user1 -file gwei_user1_ca.pub -storepass user1ca.com.cn
```



### 3.3 导出用户证书请求

```
keytool -certreq -alias user1 -file gwei_user1_ca.csr -keystore gwei_user1_ca.p12 -storepass user1ca.com.cn
```



### 3.4 用二级CA签发用户证书

```
keytool -gencert -infile gwei_user1_ca.csr -outfile subca_sign_user1_cert.crt -alias sub-ca-integration -keystore gwei_sub_ca.p12 -storepass subca.com.cn -validity 3650
```



### 3.5 导入二级公钥证书到用户秘钥库

```
keytool -importcert -file gwei_sub_ca.pub -alias sub-ca-integration -storetype PKCS12 -keystore gwei_user1_ca.p12 -storepass user1ca.com.cn
```



### 3.6 导入二级签发证书链到用户秘钥库

```
keytool -importcert -v -alias user1 -file subca_sign_user1_cert.crt -storetype PKCS12 -keystore gwei_user1_ca.p12 -storepass user1ca.com.cn
```



## 5. 其他命令

### 5.1 查看秘钥库信息

```
keytool -list -v -keystore xxx.p12 -storepass xxx
```

```
keytool -list -v  -keystore xxx.jks -storepass xxx
```

 -list 选项，命令打印证书的 MD5 指纹 

 -v 选项，将以可读格式打印证书 

 -rfc选项，将以可打印的编码格式输出证书



### 5.2 查看证书信息（CA机构、MD5、SHA、有效期）  

```
keytool -printcert -file xxx.cer
```

```
keytool -printcert -file xxx.crt
```

```
keytool -printcert -file xxx.p7r
```



### 5.3 查看证书请求CSR文件信息

```
keytool -printcertreq -rfc -file xxx.csr
```



### 5.3 删除秘钥库中的条目

```
keytool -delete -alias xxx -keystore xxx.p12 -storepass [秘钥库密码]
```



### 5.4 修改秘钥库口令

```
keytool -storepasswd -keystore xxx.p12 -storepass [原秘钥库密码]
```



### 5.5 修改秘钥库指定条目的口令

```
keytool -keypasswd -alias xxx -keystore xxx.p12 -storepass [秘钥库密码]
```



## 6 参考

- [使用keytool工具产生带根CA和二级CA的用户证书][https://blog.csdn.net/strive_or_die/article/details/98377820]
-  [Java证书工具keytool用法总结][https://blog.csdn.net/w47_csdn/article/details/87564029]