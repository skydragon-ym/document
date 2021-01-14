# WINS Local Test Server Setup



## WSL vs Virtual Machine

### WSL Pros and Cons

WSL - Windows Subsystem for Linux, ship with Windows10 (OS build 20262 or higher), share resource with Windows.

WSL is not fully implemented Linux (such as netstat, iptables command)

Host server lost network connection

> Setup jboss server in WSL but the database connection was lost very often. Finally found the Network adapter connection lost on the host server.

### Virtual Machine Pros and Cons

> Virtual Box
>
> Vmware WorkStation Player
>
> Vmware WorkStation Pro

Can use snapshot function to backup the current server setting

Need to config network for virtual machine, complex

> Host Only
>
> NAT
>
> IP Forwarding

Vmware只需要配置NAT模式即可宿主机和虚拟机互相访问

Virtual Box中NAT实现了虚拟机可以通过宿主机联网，当时宿主机仍然无法访问虚拟机，需要再配置Host Only模式，或者做端口转发

| **Mode**   | **VM→Host** | **VM←Host**                                                  | **VM1↔VM2** | **VM→Net/LAN** | **VM←Net/LAN**                                               |
| ---------- | ----------- | ------------------------------------------------------------ | ----------- | -------------- | ------------------------------------------------------------ |
| Host-only  | **+**       | **+**                                                        | **+**       | –              | –                                                            |
| Internal   | –           | –                                                            | **+**       | –              | –                                                            |
| Bridged    | **+**       | **+**                                                        | **+**       | **+**          | **+**                                                        |
| NAT        | **+**       | [Port forward](https://www.virtualbox.org/manual/ch06.html#natforward) | –           | **+**          | [Port forward](https://www.virtualbox.org/manual/ch06.html#natforward) |
| NATservice | **+**       | [Port forward](https://www.virtualbox.org/manual/ch06.html#network_nat_service) | **+**       | **+**          | [Port forward](https://www.virtualbox.org/manual/ch06.html#network_nat_service) |

### OS Version

CentOS 7 suggested for the time being

CentOS 8 not suggested, need more security setup, will not support by RedHat and replaced by CentOS stream.



## PostgreSQL Setup

### Installation

Install PostgreSQL 9.3.11 under compilation mode because there is no RPM package can use for this low version.

> yum -y install gcc gcc-c++ readline-devel zlib-devel make
>
> tar -zvxf postgresql-9.3.1.tar.gz
>
> make && make install
>
> ......



### Enhance database backup/restore scripts

Because we did VDA initial load at beginning of 2020. database size is too huge to backup and restore by the normal scripts in SQL script format. I investigated and prepared the new scripts in dump format.

> pg_dump --host segotl1230.srv.volvo.com --port 5432 --username "wins_qa" -v -Fc gwinsq01 > d:\dbquery\dump\db.dump
>
> pg_restore --host cntsnw325861 --port 5432 --username "wins_qa" -v  --create --dbname=gwinsq01 d:\dbquery\dump\db.dump



## Jboss EAP Setup

### Acquire certificate for HTTPS

#### Volvo CA

#### Self-sign certificate

#### Openshift HTTPS

Openshift has build-in HTTPS module already, don't need to do extra setup

### SSL for LDAP 

Algorithm constraints check failed on keysize limits. RSA 1024bit key used with certificate: CN=SSL_Self_Signed_Fallback

update-crypto-policies --set LEGACY

### SSL/TSL for WMQ

[MQ Configuration for JBoss - WINS - ITS Confluence Prod (volvo.net)](https://confluence.it.volvo.net/display/WINS/MQ+Configuration+for+JBoss)

#### Convert certificate file from .cdb to .jks

runmqckm -keydb -create -db wmq.jks -type jks -pw changeit

runmqckm -cert -import -db key.kdb -type cms -target wmq.jks -target_type jks -stashed

#### SSLHandshakeException - No appropriate protocol

Configuring your application to use IBM Java or Oracle Java CipherSuite mappings

| **Equivalent CipherSuite (IBM JRE)** | **Equivalent CipherSuite (Oracle JRE)** |
| ------------------------------------ | --------------------------------------- |
| SSL_RSA_WITH_AES_256_CBC_SHA256      | TLS_RSA_WITH_AES_256_CBC_SHA256         |

<property name="com.ibm.mq.cfg.useIBMCipherMappings" value="false"/>	

#### SSLHandshakeException - Channel negotiation failed

Two-way SSL/TLS, dig into SSL/TLS handshake protocol

```
Client                                               Server

      ClientHello                  -------->
                                                      ServerHello
                                                     Certificate*
                                               ServerKeyExchange*
                                              CertificateRequest*
                                   <--------      ServerHelloDone
      Certificate*
      ClientKeyExchange
      CertificateVerify*
      [ChangeCipherSpec]
      Finished                     -------->
                                               [ChangeCipherSpec]
                                   <--------             Finished
      Application Data             <------->     Application Data
```



![image-20210112163822887](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20210112163822887.png)



### JTA Transaction for WMQ 

To enable resource-adapter participate JTA transaction

 <connection-definition class-name="com.ibm.mq.connector.outbound.ManagedConnectionFactoryImpl" jndi-name="java:jboss/jms/WINS/ConnectionFactory" enabled="true" connectable="true" tracking="false" pool-name="wmq-jmsra-wins.rar">



## Performance Testing and enhancement

### Prepare script to load data to WMQ

use mqft to load file content to message queue, but preference is not good. Consider to write a tool for message batch loading.

### WMQ Transaction impacts application performance

If we setup queue on transaction mode, it will use Jboss JTA transaction manager to coordinate 2PC transaction mode. That effects performance a lot.

![image-20210111165417191](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20210111165417191.png)



### Performance on different platform

After optimize the above issue, I test message queue listener performance on different platform.

![perfermance_test](C:\Users\v0cn155\Desktop\perfermance_test.png)

## Conclusion

If there is integration involved, consider request HCL Linux first. Otherwise, local setup is better.



## Other Topics

### Podman vs Docker

#### Certificate

When pulling image from docker registry without certificate, you will get below error message:

```
docker pull fails with "x509: certificate signed by unknown authority"
```

Individual Registry Setup

Docker does have an additional location you can use to trust individual registry server CA. You can place the CA cert inside `/etc/docker/certs.d/<docker registry>/ca.crt`. Include the port number if you specify that in the image tag, e.g in Linux.

```
/etc/docker/certs.d/my-registry.example.com:5000/ca.crt
```

https://stackoverflow.com/questions/50768317/docker-pull-certificate-signed-by-unknown-authority

Global Setup

Essentially you are copying the docker registry certificate from the Services machine and placing it on workstation, master0, worker0, and worker1 and then trusting it again. You then must restart the cluster machines (master0, worker0, worker1) to get the cluster to recognize the new cert.

**scp root@services:/etc/pki/ca-trust/source/anchors/example.com.crt /etc/pki/ca-trust/source/anchors**

It's okay to overwrite the existing one - now trust it

```
update-ca-trust extract
```

#### Ignore TLS

```
podman pull mavenqa.got.volvo.net:18443/jboss-eap-7/eap72-openshift --tls-verify=false
```

![image-20201102151241632](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20201102151241632.png)



### VDA Initial Load and database function improvement

40G VDA file (240 files) need to be loaded to WINS database. 3 hours needed per file base on old design (ORM), one month in total.

Investigate jdbcTemplate. Rewrite data parse and database access layer and reuse business logic in WINS. 

20 mins per file, finished all file initial within 3 days

<img src="C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20210112121137923.png" alt="image-20210112121137923" style="zoom: 200%;" />

============================================================================================================================

![image-20210112124529615](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20210112124529615.png)

============================================================================================================================

Out of memory exception will be happened base on old design

![image-20210112122313146](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20210112122313146.png)

============================================================================================================================

![image-20210112121931049](C:\Users\v0cn155\AppData\Roaming\Typora\typora-user-images\image-20210112121931049.png)

============================================================================================================================