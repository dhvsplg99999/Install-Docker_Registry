



## Install-Docker_Regitry

[Các bước cài đặt Docker Registry ](#step)


[Cấu hình Docker kết nối đến Docker Registry trên các máy hosts](#use)


[Thực hành pull image để test kết nối](#test)



Docker Registry là nơi lưu trữ các images giống như Docker Hub nhưng với Docker Hub các images được public còn khi muốn private images thì phải trả phí cho dịch vụ này vì thế ta cần đến Docker Registry vì nó có thể lưu trữ private images .
<a name="step"></a>
### Các bước cài đặt Docker Registry 
  Cài đặt Docker Registry:

  ```
  yum install registry -y 
  ```
Configuring Secure Docker Registry và tạo SSL/TLS certificate  cho docker Registry
vì mặc định docker registry sẽ hỗ trợ https nên cần phải có SSL certificate cho docker registry. 

Registry
```
openssl req  -newkey rsa:2048  -nodes -sha256  -x509 -days 365  -keyout /etc/pki/tls/private/registry.key -out /etc/pki/tls/registry.crt
```
chúng ta sẽ tạo một private RSA key 2048 bit và key sẽ được ghi vào file ` /etc/pki/tls/private/registry.key` và file out `etc/pki/tls/registry.crt` sẽ dùng để các client xác thực với docker registry .

```
Generating a 2048 bit RSA private key
..............+++
.............................................................+++
writing new private key to '/etc/pki/tls/private/registry.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:sds-docker-registry.example.com
Email Address []:root@172.27.100.241
```

Sau khi tạo SSL certifficate chúng ta sẽ có tên miền là :
`sds-docker-registry.example.com`


cài đặt xác thưc http cho Docker registry , chúng ta sẽ dùng httpd-tools để tạo pasword.

```
yum install httpd-tools -y 
```

Khi cài đặtxong chúng ta tiến hành set up password:
```
htpasswd -c -B /etc/docker-distribution/dockerpasswd admin
```
tiến hành chỉnh sửa Docker Registry configuration file.
location of Docker Registry file : `/etc/docker-distribution/registry/config.yml`



```
version: 0.1
log:
  fields:
    service: registry
storage:
    cache:
        layerinfo: inmemory
    filesystem:
        rootdirectory: /var/lib/registry
http:
    addr: 172.27.100.241:5000
    tls:
        certificate: /etc/pki/tls/registry.crt
        key: /etc/pki/tls/private/registry.key
auth:
    htpasswd:
        realm: example.com
        path: /etc/docker-distribution/dockerpasswd
        
```
Start docker registry 
```
systemctl start docker-distribution
```
Do Docker Registry lắng nghe ở cổng 5000 nên chúng ta tiến hành mở cổng 5000 cho firewalld .
```
firewall-cmd --permanent --add-port=5000/tcp
firewall-cmd --reload
```


<a name="use"></a>
#### Cấu hình Docker kết nối đến Docker Registry trên các máy hosts:
add local DNS cho host :

```
cat >> /etc/hosts << EOF
> 172..27.100.241 sds-docker-registry.example.com 
> EOF

```
cài đặt Docker Registry Service TLS/SSL certificate trên Docker Host :

```
mkdir -p /etc/docker/certs.d/sds-docker-registry.example.com :5000
scp root@172.27.100.241:/etc/pki/tls/registry.crt /etc/docker/certs.d/sds-docker-registry.example.com\:5000/
```
<a name="test"></a>
#### Thực hành test connection đến Docker Registry

trước tiên ta sẽ pull một image từ trên Docker Hub về máy host 

``` 
docker pull jenkins
```
tiến hành gắn tag cho image
```
docker tag jenkins sds-docker-registry.example.com/jenkins 
```
Dăng nhập vào Docker Regisstry 

```
docker login sds-docker-registry.example.com:5000
```

Push  images lên Docker  Registry 

```
docker push  sds-docker-registry.example.com/jenkins
```


Nếu push  images thành công thì cho thấy đã kết nối được đến với Docker Registry .
