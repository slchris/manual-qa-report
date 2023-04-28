# Gentoo Ceph Cluster


## 环境


这里使用virtualbox作为此次的虚拟化软件

每个虚拟机有两张网卡，第一张网卡为hostonly模式用我们去管理ceph集群以及ceph的集群网络，第二张网卡为NAT用于虚拟机上网

- hostonly网卡需要设置静态ip
- NAT 网卡默认DHCP即可

这里测试的OS为：
Gentoo Linux openrc 


| hostname | ip (hostonly) | role |
| ------ | ---- | ---- | 
| ceph-node1 | 192.168.56.10 | mon，mgr, osd | 
| ceph-node2 | 192.168.56.11 | mon，mgr, osd | 
| ceph-node2 | 192.168.56.11 | mon，mgr, osd | 
| gentoo-client | 192.168.56.20 | client | 

除此之外还有一台额外的机器作为`binpkg`服务器我们将会在这个服务器上构建ceph的二进制包并在上面启动web服务器用于binpkg的分发


配置静态ip的参考文档： https://wiki.gentoo.org/wiki/Netifrc

> ceph-node1

创建并编辑`/etc/conf.d/net` 文件，內容如下：


```
cat << EOF > /etc/conf.d/net
# For a static IP using CIDR notation
config_enp0s3="192.168.56.10/24"
config_enp0s8="dhcp"
dns_servers_enp0s8="114.114.114.114"
EOF
```

创建服务软连接：


```shell
ln -svf /etc/init.d/net.lo /etc/init.d/net.enp0s3
ln -svf /etc/init.d/net.lo /etc/init.d/net.enp0s8
```

将其加入到开机启动：

```
rc-update add net.enp0s3 default
rc-update add net.enp0s8 default
```

确保已经将dhcpcd或者是networkmanager之类的服务关闭掉：

```shell
rc-update delete dhcpcd default
```

设置主机名：


```
vi /etc/conf.d/hostname
```

配置`/etc/hosts` 添加内容如下：


```shell
192.168.56.10 ceph-node1
192.168.56.11 ceph-node2
192.168.56.12 ceph-node3
```


其他的节点也是类似的操作。

### 时间同步服务器

在集群中时间同步是非常重要的一部分 这里我们需要安装和配置一下NTP

```shell
emerge --ask net-misc/ntp
```

配置ntp
编辑`/etc/conf.d/ntp-client`文件，修改内容如下：

```
NTPCLIENT_CMD="ntpdate"
NTPCLIENT_OPTS="-s -b -u \
	ntp.aliyun.com"
```

启动ntp服务：

```shell
rc-service ntp-client start
```
加入开机启动

```shell
rc-update add ntp-client default
```

查看4个节点的ip是否一样：

```shell
date
```



## binpkg server


### 设置服务端

这里使用的是gpkg格式的binpkg server，需要一个gpg key（用于给包做签名） 同時还需要一个http服务器来去发布编译好的二进制包。


`make.conf`的內容如下：

```
FEATURES="buildpkg binpkg-signing gpg-keepalive"

BINPKG_GPG_SIGNING_GPG_HOME="/etc/portage/gnupg"
BINPKG_GPG_SIGNING_KEY="0x45D10727855239D3"
BINPKG_GPG_VERIFY_GPG_HOME="/etc/portage/gnupg"
BINPKG_FORMAT="gpkg"
GPG_VERIFY_USER_DROP="portage"
GPG_VERIFY_GROUP_DROP="portage"
```

`BINPKG_GPG_SIGNING_KEY`这个key就是生成的gpg key，这部分的教程可以参考:https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key

构建整个系统：

```shell
emerge -ueDN @world
```

构建ceph:

```shell
emerge -av ceph
```


制作二进制包：

```shell
quickpkg --include-config=y sys-cluster/ceph
```

`--include-config` 这个选项需要是y 不然openrc下面会缺少配置


> 注意这一步需要使用root用戶


发布binpkgserver：

```shell
cd /var/cache/binpkgs
python -m http.server 9000
```

### 节点使用binpkgserver

创建一个生成和导入gpg key的脚本`create-portage-local-gpg`，內容如下：

```shell
#!/bin/bash
 
KEY_NAME="sa-slchris.asc"
LOCAL_KEYSERVER="http://192.168.56.109:9001"
KEY="5BB4DC20DCA50A499219935036F2D646E2E89820" # Key fingerprint
 
GPG_DIR="/etc/portage/gnupg"
 
PASS="$(openssl rand -base64 32)"
 
KEY_CONFIG_FILE="$(mktemp)"
chmod 600 "${KEY_CONFIG_FILE}"
 
export GNUPGHOME="${GPG_DIR}"
cat > "${KEY_CONFIG_FILE}" <<EOF
     %echo Generating Portage local OpenPGP trust key
     Key-Type: default
     Subkey-Type: default
     Name-Real: Portage Local Trust Key
     Name-Comment: local signing only
     Name-Email: portage@localhost
     Expire-Date: 0
     Passphrase: ${PASS}
     %commit
     %echo done
EOF
 
mkdir -p "${GNUPGHOME}"
gpg --batch --generate-key "${KEY_CONFIG_FILE}"
rm -f "${KEY_CONFIG_FILE}"
 
touch "${GNUPGHOME}/pass"
chmod 600 "${GNUPGHOME}/pass"
echo "${PASS}" > "${GNUPGHOME}/pass"

curl  $LOCAL_KEYSERVER/$KEY_NAME | gpg --import - 
gpg --batch --yes --pinentry-mode loopback --passphrase "${PASS}" --sign-key "${KEY}" 
echo -e "5\ny\n" | gpg --command-fd 0 --edit-key "${KEY}" trust
 
chmod ugo+r "${GNUPGHOME}/trustdb.gpg"
```

执行：

```shell
bash create-portage-local-gpg
```

执行完成之后可以看到对应的key：

![](./images/gpg-list-keys.png)

集群节点编辑`make.conf`文件添加內容如下：

```shell
EMERGE_DEFAULT_OPTS="${EMERGE_DEFAULT_OPTS} --getbinpkgonly"
FEATURES="getbinpkg binpkg-request-signature"
PORTAGE_BINHOST="http://192.168.56.109:9000"
```


在每个节点上安装ceph:
```shell
emerge -av ceph
```

## 配置CEPH集群


### MON 配置


在gentoo里面`/etc/ceph`文件夹没有被默认创建（可以考虑后续优化一下ebuild），需要手动创建一下：

```shell
mkdir -pv /etc/ceph
```
生成一个uuid作为此次集群使用：

```shell
uuidgen
```

修改`/etc/conf.d/ceph`文件 取消读取配置文件的注释：

```shell
ceph_conf="/etc/ceph/ceph.conf"
```

> ceph-node1 上操作


创建并编辑`/etc/ceph/ceph.conf` 文件内容如下：

```shell
[global]
fsid = 333c6f6c-124d-4410-bde3-fb6da688dea9 # 使用uuidgen的
mon initial members = mon.a, mon.b, mon.c
mon host = 192.168.56.10:6789, 192.168.56.11:6789, 192.168.56.12:6789
public network = 192.168.56.0/24 # 公开的网络
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
osd journal size = 1024
osd pool default size = 3
osd pool default min size = 2
osd pool default pg num = 333
osd pool default pgp num = 333
osd crush chooseleaf type = 1
```
创建客户端的admin keyring：

```shell
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
```


创建mon的keyring：

```shell
ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

导入客户端mon的keyring：

```shell
ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```


创建osd的bootstrap文件夹：
```shell
mkdir -pv /var/lib/ceph/bootstrap-osd/
```

```shell
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
```
合并key：

```shell
cd /etc/ceph
cat /var/lib/ceph/bootstrap-osd/ceph.keyring >>ceph.mon.keyring
```
使用主机名，ip地址和集群uuid去生成mon map

```shell
cd /etc/ceph
monmaptool --create --add mon.a 192.168.56.10 --add mon.b 192.168.56.11 --add mon.c 192.168.56.12 --fsid 333c6f6c-124d-4410-bde3-fb6da688dea9 initial-monmap
```

同步文件：

```shell
rsync -av /etc/ceph/ 192.168.56.11:/etc/ceph/
rsync -av /etc/ceph/ 192.168.56.12:/etc/ceph/
```


创建mon文件夹：

```shell
mkdir -p /var/lib/ceph/mon/ceph-mon.a
chown -R ceph:ceph /var/lib/ceph
```

清理日志：

```shell
cd /var/log/ceph
rm -rf *
```

初始化：
```shell
cd /etc/ceph
ceph-mon --mkfs -i mon.a --monmap initial-monmap --keyring ceph.mon.keyring
```

查看文件初始化情况：
```shell
ls -l /var/lib/ceph/mon/ceph-mon.a
```


修改目录权限：

```shell
chown -R ceph:ceph /var/lib/ceph/mon/ceph-mon.a
```


> openrc

添加软连接：
```shell
ln -s /etc/init.d/ceph /etc/init.d/ceph-mon.mon.a
```

启动服务：
```shell
/etc/init.d/ceph-mon.mon.a start
```

加入开机启动：

```shell
rc-update add ceph-mon.mon.a default
```

> systemd 

在启动mon之前需要对默认的systemd service文件进行修改

```shell
vi /lib/systemd/system/ceph-mon@.service
```

修改`MemoryDenyWriteExecute`的值为`false` 这个具体原因可以看 https://bugs.launchpad.net/ubuntu/+source/ceph/+bug/1917414


修改完成之后:

```shell
systemctl daemon-reload
```

启动mon:

```shell
systemctl start ceph-osd@gentoo 
```

加入开机启动（可选）：

```shell
systemctl enable ceph-osd@gentoo 
```

查看运行状态：
```shell
systemctl status ceph-osd@gentoo 
```

> ceph-node2

node2 节点上操作如下：

```shell
mkdir -p /var/lib/ceph/mon/ceph-mon.b
cd /etc/ceph
ceph-mon --mkfs -i mon.b --monmap initial-monmap --keyring ceph.mon.keyring
chown -R ceph:ceph /var/lib/ceph/mon
cd /etc/init.d
ln -s ceph ceph-mon.mon.b
./ceph-mon.mon.b start
rc-update add ceph-mon.mon.b default
```

> ceph-node3

node3节点上操作如下：

```shell
mkdir -p /var/lib/ceph/mon/ceph-mon.c
cd /etc/ceph
ceph-mon --mkfs -i mon.c --monmap initial-monmap --keyring ceph.mon.keyring
chown -R ceph:ceph /var/lib/ceph/mon
cd /etc/init.d
ln -s ceph ceph-mon.mon.c
./ceph-mon.mon.c start
rc-update add ceph-mon.mon.c default
```


验证：

```shell
ceph -s 
```

在services的部分如果可以看到有三个mon已经启动了 就说明配置成功了

```
  cluster:
    id:     333c6f6c-124d-4410-bde3-fb6da688dea9
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            3 monitors have not enabled msgr2
 
  services:
    mon: 3 daemons, quorum mon.a,mon.b,mon.c (age 94s)
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 

```

就说明mon已经启动成功了。





### 配置mgr



创建mgr对应的目录：

```shell
name=a
mkdir -pv /var/lib/ceph/mgr/ceph-$name # 这里的ceph是集群的名称
```

生成mgr的keyring：

```shell
ceph auth get-or-create mgr.$name mon 'allow profile mgr' osd 'allow *' mds 'allow *' >>  /var/lib/ceph/mgr/ceph-$name/keyring 
```

修改权限：

```shell
chown -R ceph:ceph /var/lib/ceph/mgr
```

> openrc

添加软连接：
```shell
cd /etc/init.d
ln -s ceph ceph-mgr.a
```

启动：
```shell
/etc/init.d/ceph-mgr.a start
```

开机启动：

```shell
rc-update add ceph-mgr.a default
```


启动对应的mgr（这里好像是没有直接启动，还是要systemd来启动）：
```shell
ceph-mgr -i $name
```


使用systemd启动之前需要先将`/lib/systemd/system/ceph-mgr@.service` 配置文件中的`MemoryDenyWriteExecute` 设置为`false`



```shell
vi /lib/systemd/system/ceph-mgr@.service
```

修改完成之后运行：

```shell
systemctl daemon-reload
```

启动服务：

```shell
systemctl start ceph-mgr@gentoo.service
```


> ceph-node2


```shell
name=b
mkdir -pv /var/lib/ceph/mgr/ceph-$name # 这里的ceph是集群的名称
```

生成mgr的keyring：

```shell
ceph auth get-or-create mgr.$name mon 'allow profile mgr' osd 'allow *' mds 'allow *' >>  /var/lib/ceph/mgr/ceph-$name/keyring 
```

修改权限：

```shell
chown -R ceph:ceph /var/lib/ceph/mgr
```

> openrc

添加软连接：
```shell
cd /etc/init.d
ln -s ceph ceph-mgr.b
```

启动：
```shell
/etc/init.d/ceph-mgr.b start
```

开机启动：

```shell
rc-update add ceph-mgr.b default
```

> ceph-node3

```shell
name=c
mkdir -pv /var/lib/ceph/mgr/ceph-$name # 这里的ceph是集群的名称
```

生成mgr的keyring：

```shell
ceph auth get-or-create mgr.$name mon 'allow profile mgr' osd 'allow *' mds 'allow *' >>  /var/lib/ceph/mgr/ceph-$name/keyring 
```

修改权限：

```shell
chown -R ceph:ceph /var/lib/ceph/mgr
```

> openrc

添加软连接
```shell
cd /etc/init.d
ln -s ceph ceph-mgr.c
```

启动：
```shell
/etc/init.d/ceph-mgr.c start
```

开机启动：

```shell
rc-update add ceph-mgr.c default
```

验证：

```shell
ceph -s
```



在services的部分可以看到mgr已经启动了：
```shell
  cluster:
    id:     333c6f6c-124d-4410-bde3-fb6da688dea9
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            3 monitors have not enabled msgr2
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum mon.a,mon.b,mon.c (age 6m)
    mgr: a(active, since 3m), standbys: b, c
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 
```

并且是有一个为主，还有两个是standbys


### 配置osd


> bluestore

确保`/etc/lvm/`下的配置是正确的



```shell
cp -v /var/lib/ceph/bootstrap-osd/ceph.keyring /etc/ceph/
```

创建osd的目录：
```shell
mkdir -pv /var/lib/ceph/osd
```

创建对应的 osd

```shell
ceph-volume lvm create --no-systemd --bluestore  --data /dev/sdb
```


备份：

```shell
mkdir -pv backup
cd backup
cp -rv  /var/lib/ceph/osd/ceph-*/* .
```

卸载：
```shell
umount -R /var/lib/ceph/osd/ceph-* 
mv * /var/lib/ceph/osd/ceph-0/
```


修改权限：

```shell
chown -R  ceph: /var/lib/ceph/osd
```



创建服务
```shell
ln -sf /etc/init.d/ceph /etc/init.d/ceph-osd.0
```
启动服务：

```shell
/etc/init.d/ceph-osd.0 start
```

开机启动：

```shell
rc-update add ceph-osd.0 default
```


同步文件到其他的节点：
```shell
rsync -av /etc/ceph/ 192.168.56.11:/etc/ceph/
rsync -av /etc/ceph/ 192.168.56.12:/etc/ceph/
rsync -av /var/lib/ceph/bootstrap-osd/ 192.168.56.11:/var/lib/ceph/bootstrap-osd/
rsync -av /var/lib/ceph/bootstrap-osd/ 192.168.56.12:/var/lib/ceph/bootstrap-osd/
rsync -av /etc/lvm/ 192.168.56.11:/etc/lvm/
rsync -av /etc/lvm/ 192.168.56.12:/etc/lvm/
```

> ceph-node2




创建osd的目录：
```shell
mkdir -pv /var/lib/ceph/osd
```

创建对应的 osd

```shell
ceph-volume lvm create --no-systemd --bluestore  --data /dev/sdb
```

修改权限：

```shell
chown ceph: /var/lib/ceph/osd/ceph-* -R
```

备份：

```shell
mkdir -pv backup
cd backup
cp -rv  /var/lib/ceph/osd/ceph-*/* .
```

卸载：
```shell
umount -R /var/lib/ceph/osd/ceph-* 
mv * /var/lib/ceph/osd/ceph-1/
```


修改权限：

```shell
chown -R  ceph: /var/lib/ceph/osd
```



创建服务
```shell
ln -sf /etc/init.d/ceph /etc/init.d/ceph-osd.1
```
启动服务：

```shell
/etc/init.d/ceph-osd.1 start
```

开机启动：

```shell
rc-update add ceph-osd.1 default
```


> ceph-node3

创建osd的目录：
```shell
mkdir -pv /var/lib/ceph/osd
```

创建对应的 osd

```shell
ceph-volume lvm create --no-systemd --bluestore  --data /dev/sdb
```


修改权限：

```shell
chown ceph: /var/lib/ceph/osd/ceph-* -R
```


备份：

```shell
mkdir -pv backup
cd backup
cp -rv  /var/lib/ceph/osd/ceph-*/* .
```

卸载：
```shell
umount -R /var/lib/ceph/osd/ceph-* 
mv * /var/lib/ceph/osd/ceph-2/
```


修改权限：

```shell
chown -R  ceph:ceph /var/lib/ceph/osd
```



创建服务
```shell
ln -sf /etc/init.d/ceph /etc/init.d/ceph-osd.2
```
启动服务：

```shell
/etc/init.d/ceph-osd.2 start
```

开机启动：

```shell
rc-update add ceph-osd.2 default
```

> systemd 

在启动osd之前需要先修改一下`/lib/systemd/system/ceph-osd@.service` 文件：

```shell
vi /lib/systemd/system/ceph-osd@.service
```

将`MemoryDenyWriteExecute`的值修改为`false`



然后运行：

```shell
systemctl daemon-reload
```


启动osd服务：
```shell
systemctl start ceph-osd@gentoo.service
```


验证：
```shell
ceph -s
```



看到service 的部分有osd启动即可：
```shell
  cluster:
    id:     333c6f6c-124d-4410-bde3-fb6da688dea9
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            3 monitors have not enabled msgr2
 
  services:
    mon: 3 daemons, quorum mon.a,mon.b,mon.c (age 6m)
    mgr: a(active, since 6m), standbys: b, c
    osd: 3 osds: 3 up (since 51s), 3 in (since 81s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 24 GiB / 24 GiB avail
    pgs:     1 active+clean
    
```

这里可以看到有3个osd 且可用的空间为 8  * 3

查看osd tree:

```shell
ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT  PRI-AFF
-1         0.02339  root default                                  
-3         0.00780      host ceph-node1                           
 0    hdd  0.00780          osd.0            up   1.00000  1.00000
-5         0.00780      host ceph-node2                           
 1    hdd  0.00780          osd.1            up   1.00000  1.00000
-7         0.00780      host ceph-node3                           
 2    hdd  0.00780          osd.2            up   1.00000  1.00000

```



## 元数据服务器

在主配置文件中添加：

```shell
[mds.a]
        host = ceph-node1
```


创建文件夹：

```shell
mkdir -p /var/lib/ceph/mds/ceph-a
```
创建key：
```shell
ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-a/keyring --gen-key -n mds.a
```
创建keyring:
```shell
ceph auth add mds.a osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-a/keyring
```

修改权限：
```shell
chown -R ceph:ceph /var/lib/ceph/mds
```


```shell
cd /etc/init.d
ln -s ceph ceph-mds.a
rc-update add ceph-mds.a default
./ceph-mds.a start
```


```shell
ceph mds stat
```


## 测试功能



### dashboard



开启dashboard：

```shell
ceph mgr module enable dashboard
```

关闭ssl：

```shell
ceph config set mgr mgr/dashboard/ssl false
```

设置服务器ip地址：
```shell
ceph config set mgr.a mgr/dashboard/server_addr 192.168.56.10 # 这里替换成你的ip
```



生成密码文件:

```shell
uuidgen > ceph-admin-password.txt
```



创建管理员用户：

```shell
ceph dashboard ac-user-create admin -i  ceph-admin-password.txt administrator
```



验证：

打开浏览器输入：http://ip:8080  ip替换成你的ip，如果一切正常就可以看到ceph的dashboard登陆页面：



![](./images/ceph-login.png)



输入设置好的账号密码就可以看到ceph的dashboard了：



![](./images/ceph-dashboard.png)





### cephfs

创建pool：

```shell
ceph osd pool create cephfs_data 1
```
创建元数据的pool：

```shell
ceph osd pool create cephfs_metadata 64
```
创建cephfs：

```shell
ceph fs new cephfs cephfs_metadata cephfs_data
```

查看cephfs：

```shell
ceph fs ls
```

查看元数据服务的状态：

```shell
ceph mds stat
```


> 客户端使用 挂载cephfs

查看admin的keyring（这里只是为了做验证测试用的admin实际生产使用请生成对应的keyring）


```shell
[client.admin]
	key = AQDMCDdkU31DHBAAaft6xc9KFekiQahkq1sHAA==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"
```

创建一个挂载点：

```shell
mkdir -pv /test
```

修改fstab文件（开机挂载），添加内容如下：


```shell
192.168.56.10:/	/test	ceph		mds_namespace=cephfs,name=admin,secret=AQDMCDdkU31DHBAAaft6xc9KFekiQahkq1sHAA==	0 0
```

测试挂载：
```shell
mount -t ceph 192.168.56.10:/ /test -o mds_namespace=cephfs,name=admin,secret=AQDMCDdkU31DHBAAaft6xc9KFekiQahkq1sHAA==
```

验证：

```shell
df -Th
```


```shell
dd if=/dev/zero of=test.img bs=4M count=1024 status=progress
```



### 对象存储（测试中）



在`ceph-node`这个节点上来配置 radosgw

在`/etc/ceph/ceph.conf`下添加如下内容：

```shell
[client.rgw.ceph-node1]
     host = ceph-node1
     rgw frontends = "civetweb port=80"
     rgw dns name = ceph-node1.homelab.sh
```

创建文件夹：

```shell
mkdir -p /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`
```

创建keyring：
```shell
ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring
ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.gateway --gen-key
ceph-authtool -n client.radosgw.gateway --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.gateway -i /etc/ceph/ceph.client.radosgw.keyring
```
创建凭证：
```shell
ceph auth get-or-create client.rgw.`hostname -s` osd 'allow rwx' mon 'allow rw' -o /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`/keyring
```
创建done这个文件：

```shell
touch /var/lib/ceph/radosgw/ceph-rgw.`hostname -s`/done
```

修改权限：

```shell
chown -R ceph:ceph /var/lib/ceph/radosgw
chown -R ceph:ceph /var/log/ceph
chown -R ceph:ceph /var/run/ceph
chown -R ceph:ceph /etc/ceph
```

启动服务：

```shell
ln -sf /usr/bin/radosgw   /usr/bin/ceph-radosgw
```


> 验证




创建文件夹：
```shell
mkdir -p /var/lib/ceph/radosgw/<cluster_name>-rgw.`hostname -s`
```



### 块存储


创建`libvirt-pool`:

```shell
ceph osd pool create libvirt-pool 128 128
```

创建完成之后查看pool是否存在：
```shell
ceph osd lspools
```

使用`rbd`初始化pool:
```shell
rbd pool init libvirt-pool
```


创建一个凭证，这个凭证用于libvirt去使用我们创建的`libvirt-pool`：
```shell
ceph auth get-or-create client.libvirt mon 'profile rbd' osd \
 'profile rbd pool=libvirt-pool'
```


查看是否存在：
```shell
ceph auth list
```



> 客户端配置


这里使用libvirt，需要启动对应的USE：

```shell
USE="rbd" emerge -v libvirt
```

为了能够正常使用ceph rbd 我们还需要配置一下libvirt服务。


编辑`/etc/libvirt/libvirtd.conf`文件，修改以下内容：
```shell
auth_unix_ro = "none"
auth_unix_rw = "none"
unix_sock_group = "libvirt"
unix_sock_ro_perms = "0777"
unix_sock_rw_perms = "0770"
```

启动服务并加入开机启动：
```shell
rc-service libvirtd start && rc-update add libvirtd default
```

准备一个centos的iso：
```shell
wget -c https://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2009.iso
```



拷贝`/etc/ceph/ceph.conf`到客户端：
```shell
scp /etc/ceph/ceph.conf 192.168.56.20:/etc/ceph/ceph.conf
```


```shell
ceph auth get-key client.libvirt | tee /etc/ceph/client.libvirt.key
```

拷贝对应的keyring到当前节点：
```shell
scp /etc/ceph/ceph.client.libvirt.keyring 192.168.56.20:/etc/ceph/
scp /etc/ceph/client.libvirt.key 192.168.56.20:/etc/ceph/
```



创建盘：
```shell
qemu-img create -f raw rbd:libvirt-pool/new-libvirt-image:id=libvirt 2G
```

查看硬盘是否存在：
```shell
rbd -p libvirt-pool ls
```

生产使用的话还是创建pool的方式比较好，使用pool的话需要县创建一个secret：
```shell
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
        <usage type='ceph'>
                <name>client.libvirt secret</name>
        </usage>
</secret>
EOF
```
创建：
```shell
sudo virsh secret-define --file secret.xml
```
设置对应的secret：
```shell
virsh secret-set-value --secret 0bb9f696-12f5-4665-be2c-cc26966614ee --base64 $(cat /etc/ceph/client.libvirt.key) 
```

创建rbd的存储卷：
```shell
cat > libvirt-rbd-pool.xml <<EOF
<pool type="rbd">
  <name>libvirt-pool</name>
  <source>
    <name>libvirt-pool</name>
    <host name='192.168.56.10' port='6789'/>
    <host name='192.168.56.11' port='6789'/>
    <host name='192.168.56.12' port='6789'/>
	<name>libvirt-pool</name>
    <auth username='libvirt' type='ceph'>
      <secret uuid='0bb9f696-12f5-4665-be2c-cc26966614ee'/>
    </auth>
  </source>
</pool>
EOF
virsh pool-define libvirt-rbd-pool.xml
```

查看所有的pool：
```shell
virsh pool-list --all
```
启动：
```shell
virsh pool-start libvirt-pool
```

加入开机启动：
```shell
virsh pool-autostart  libvirt-pool
```

我们可以在pool里面创建一个硬盘来验证是否成功，在后面创建虚拟机也会使用到这个pool：
```shell
virsh vol-create-as libvirt-pool centos7 --capacity 20G --format raw
```
查看创建的卷：
```shell
virsh vol-list libvirt-pool
```


使用virt-install创建虚拟机，首先需要安装virt-manager:
```shell
emerge -v virt-manager dev-libs/libisoburn
```
dev-libs/libisoburn 是使用iso安装需要的一个库

```shell
mkdir -pv /var/lib/libvirt/boot /opt/qemu/iso
```

修改权限：
```shell
chmod 777 /opt/qemu/ 
```
生产不推荐这么做，最好是改成libvirt有权限的。


创建虚拟机：
```shell
virt-install \
--force \
--name=centos7 \
--memory=2048 \
--vcpus=2 \
--location=/opt/qemu/iso/CentOS-7-x86_64-Minimal-2009.iso \
--disk vol=libvirt-pool/centos7 \
--accelerate \
--network network=default,model=virtio \
--nographics \
--extra-args console=ttyS0
```

这里可以看到磁盘


可以使用以下命令持续查看ceph io的情况：
```shell
ceph -w
```


### 监控


这里使用prometheus进行ceph的监控





## 修复警告


reclaim警告：
```shell
mons are allowing insecure global_id reclaim
```

在每个mon节点上执行：

```shell
ceph config set mon mon_warn_on_insecure_global_id_reclaim true
ceph config set mon mon_warn_on_insecure_global_id_reclaim_allowed true
ceph config set mon auth_allow_insecure_global_id_reclaim false
```


> 3 monitors have not enabled msgr2

每个mon节点上执行：

```shell
ceph mon enable-msgr2
```




## TODO List





- 整理patch 
- openrc 下 ceph osd 每次启动block都会报没有权限 需要重新授权之后才能够正常启动 这个可能需要udev规则或者是openrc service来进一步配置 (目前简单的解决方式就是在重启机器之后 `chown  ceph:ceph /var/lib/ceph/osd/ceph-*/block && /etc/init.d/ceph-osd.* restart `
)
- bluestore 测试 已经通过
- openrc radosgw 启动有问题 无法正常读取到mon的配置文件以及keyring
- systemd 需要修改`MemoryDenyWriteExecute`的值这部分需要单独的patch 
