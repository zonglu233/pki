# 帮助文档

---

## 目录结构
 - `/etc`配置文件文件夹
 - `/ca`用来存储CA请求和数据库
 - `/certs`签发的用户证书文件夹
 - `/crl`吊销列表文件夹

---

## 操作说明

### 1. 创建根证书

#### 1.1 初始化目录结构
  首先，进入项目的根路径。
  新建根证书的存储目录，并创建根证书存储数据库
  ```
mkdir -p ca/root-ca/private ca/root-ca/db crl/root-ca
chmod 700 ca/root-ca/private

cp /dev/null ca/root-ca/db/root-ca.db
cp /dev/null ca/root-ca/db/root-ca.db.attr
echo 01 > ca/root-ca/db/root-ca.crt.srl
echo 01 > ca/root-ca/db/root-ca.crl.srl
  ```

#### 1.2 请求签发根证书
  首先，修改`/etc/root-ca.conf`文件的配置属性。然后执行请求命令
  ```
  openssl req -new -config etc/root-ca.conf -out ca/root-ca.csr -keyout ca/root-ca/private/root-ca.key
  ```
  输入PEM的参数密码，例如：s***s(A...3)

#### 1.2 签发根证书
  使用自签名命令，根据请求的csr文件，以及配置文件，执行签名命令，生成crt证书
  ```
  openssl ca -selfsign -config etc/root-ca.conf -in ca/root-ca.csr -out ca/root-ca.crt -extensions root_ca_ext
  ```

#### 1.3 初始化吊销列表
  执行吊销证书的更新操作（吊销证书更新应每周或每天进行，待进一步研究）
  ```
  openssl ca -gencrl -config etc/root-ca.conf -out crl/root-ca/root-ca.crl
  ```
  （需要输入请求证书使用的PEM密码）

### 2. 创建中间签发证书

#### 2.1 初始化
  新建中间证书的存储目录，并创建中间证书存储数据库
  例如，创建Sign的中间签发证书
  ```
mkdir -p ca/signing-ca/private ca/signing-ca/db crl certs
chmod 700 ca/signing-ca/private

cp /dev/null ca/signing-ca/db/signing-ca.db
cp /dev/null ca/signing-ca/db/signing-ca.db.attr
echo 01 > ca/signing-ca/db/signing-ca.crt.srl
echo 01 > ca/signing-ca/db/signing-ca.crl.srl
  ```

#### 2.2 请求
  首先，复制 `/signing-ca.conf` 文件，根据工作区，新建配置文件，并配置其属性，例如ebeiport-ca.conf。然后执行请求命令
  ```
  openssl req -new -config etc/signing-ca.conf -out ca/signing-ca.csr -keyout ca/signing-ca/private/signing-ca.key
  ```
  输入PEM的参数密码，例如：hebei(A...3)

#### 2.3 签发
  使用根证书的配置信息签发
  ```
  openssl ca -config etc/root-ca.conf -in ca/signing-ca.csr -out ca/signing-ca.crt -extensions signing_ca_ext
  ```

#### 2.4 初始化吊销
  ```
openssl ca -gencrl -config etc/signing-ca.conf -out crl/signing-ca.crl
  ```
#### 2.5 创建PEM包
  连锁在一起的
  ```
cat ca/signing-ca.crt ca/root-ca.crt > ca/signing-ca-chain.pem
  ```

### 3. 签发用户证书

#### 3.1 请求(以jbws证书为例)
  根据不同用户，修改 `signing.conf` 配置信息，执行命令
  ```
  openssl req -new -config etc/signing.conf -out certs/jbws.csr -keyout certs/jbws.key
  ```
  （依次填写相应信息：CN - Hebei - Qinhuangdao - Steedos Inc. - Steedos Inc. - JBWS - jbws@portqhd.com）
  其中，organizationName必须与根证书配置文件相同，即Steedos Inc.

#### 3.2 创建
  根据中间证书，签发用户证书，用户证书为crt格式，用户名可自定义，例如jbws
  ```
  openssl ca -config etc/signing-ca.conf -in certs/jbws.csr -out certs/jbws.crt -extensions signing_ext
  ```

#### 3.3 初始化吊销CRL
  ```
  openssl ca -gencrl -config etc/signing-ca.conf -out crl/signing-ca.crl
  ```

#### 3.4 创建PKCS＃12证书
  ```
  openssl pkcs12 -export -name "JBWS" -caname "HeBei Port Group" -caname "Steedos Root Certificate Authority" -inkey certs/jbws.key -in certs/jbws.crt -certfile ca/signing-ca-chain.pem -out certs/jbws.p12
  ```

#### 3.5 吊销证书
  如果use key丢失，可以通过该命令将证书吊销
  ```
  openssl ca -config etc/signing-ca.conf -revoke ca/signing-ca/01.pem -crl_reason superseded
  ```
  证书吊销以后，一定要再次更新吊销列表
  ```
  openssl ca -gencrl -config etc/signing-ca.conf -out crl/signing-ca.crl
  ```


#### 附加：创建股份公司文书证书
  ```

  openssl req -new -config etc/signing.conf -out certs/gfgsws.csr -keyout certs/gfgsws.key

  openssl ca -config etc/signing-ca.conf -in certs/gfgsws.csr -out certs/gfgsws.crt -extensions signing_ext

  openssl ca -gencrl -config etc/signing-ca.conf -out crl/signing-ca.crl

  openssl pkcs12 -export -name "GFGSWS" -caname "HeBei Port Group" -caname "Steedos Root Certificate Authority" -inkey certs/gfgsws.key -in certs/gfgsws.crt -certfile ca/signing-ca-chain.pem -out certs/gfgsws.p12

  ```

  CN - Hebei - Qinhuangdao - Steedos Inc. - Steedos Inc. - GFGSWS - gfgsws@portqhd.com


#### 附加：创建oscp证书
  ```
  openssl req -new -config etc/ocspsign.conf -out certs/ocsp.csr -keyout certs/ocsp.key

  openssl ca -config etc/component-ca.conf -in certs/ocsp.csr -out certs/ocsp.crt -extensions ocspsign_ext -days 14

  ```

---

#### 输出格式
1.创建DER证书
openssl x509 -in certs/fred.crt -out certs/fred.cer -outform der

2.创建DER
openssl crl -in crl/hebeiport-ca.crl -out crl/hebeiport-ca.crl -outform der

3.创建PKCS＃7软件包
openssl crl2pkcs7 -nocrl -certfile ca/hebeiport-ca.crt -certfile ca/root-ca.crt -out ca/hebeiport-ca-chain.p7c -outform der

4.创建PKCS＃12
openssl pkcs12 -export -name "JBWS" -inkey certs/jbws.key -in certs/jbws.crt -out certs/jbws.p12

5.创建PEM包
cat ca/hebeiport-ca.crt ca/root-ca.crt > ca/hebeiport-ca-chain.pem

cat certs/fred.key certs/fred.crt > certs/fred.pem



#### 查看
1.查看请求
openssl req -in certs/fred.csr -noout -text

2.查看证书
openssl x509 -in certs/fred.crt -noout -text

3.查看CRL
openssl crl -in crl/hebeiport-ca.crl -inform der -noout -text

4.查看PKCS＃7束
openssl pkcs7 -in ca/hebeiport-ca-chain.p7c -inform der -noout -text -print_certs

5.查看PKCS＃12
openssl pkcs12 -in certs/fred.p12 -nodes -info




#### Component证书
1.创建
mkdir -p ca/component-ca/private ca/component-ca/db crl certs
chmod 700 ca/component-ca/private

2.创建数据库
cp /dev/null ca/component-ca/db/component-ca.db
cp /dev/null ca/component-ca/db/component-ca.db.attr
echo 01 > ca/component-ca/db/component-ca.crt.srl
echo 01 > ca/component-ca/db/component-ca.crl.srl

3.创建CA请求
openssl req -new -config etc/component-ca.conf -out ca/component-ca.csr -keyout ca/component-ca/private/component-ca.key

4.创建证书
openssl ca -config etc/root-ca.conf -in ca/component-ca.csr -out ca/component-ca.crt -extensions signing_ca_ext

5.初始化
openssl ca -gencrl -config etc/component-ca.conf -out crl/component-ca.crl

6.创建PEM包
cat ca/component-ca.crt ca/root-ca.crt > ca/component-ca-chain.pem


#### OCSP证书
1.请求
openssl req -new -config etc/ocspsign.conf -out certs/ocsp1.csr -keyout certs/ocsp1.key

2.颁发
openssl ca -config etc/component-ca.conf -in certs/ocsp1.csr -out certs/ocsp1.crt -extensions ocspsign_ext -days 1

3.吊销
openssl ca -config etc/component-ca.conf -revoke ca/component-ca/XXXXXX.pem -crl_reason superseded

4.初始化
openssl ca -gencrl -config etc/component-ca.conf -out crl/component-ca.crl

5.生成证书
openssl pkcs12 -export -name "OCSP1" -inkey certs/ocsp1.key -in certs/ocsp1.crt -out certs/ocsp1.p12



------
Tips:需要使用git bush和cmd一起操作，linux命令使用git hub，而openssl的操作使用cmd