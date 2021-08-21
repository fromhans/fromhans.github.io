defaults:
  # _posts
  - scope:
      path: ""
      type: posts
    values:
      layout: single
      author_profile: true
      read_time: true
      comments: true
      share: true
      related: true
# **HA 제공을 위한 Keepalived & DRBD 적용**

## **ㅇ테스트환경**

- OS: Centos 7.6
- Host 정보
    - Master Node : 10.30.31.168/27
    - Backup Node : 10.30.31.170/27
    - Ping/ARP Test용 Node : 10.30.31.172
    - VIP : 10.30.31.171/27

## **ㅇ상세 작업 내용**

### **1. keepalived 프레임워크 설치**

- Master/Backup 모두에서 실행
    - 상용 서버의 /root/pkgs/keepalived 경로 내용을 구축대상 서버의 동일경로로 복사

```bash
#yum 명령어는 상용 서버에서 정상동작 하지 않으므로 아래와 같이 수행
cd /root/pkgs/keepalived

```

- keepalived 설치

```bash
yum install -y glibc
rpm -ivh ipset-libs<TAB>
rpm -ivh net-snmp-libs<TAB> net-snmp-agent<TAB>
rpm -ivh keepalived<TAB>
```

### 2. **Master Node에 Keepalived config 적용**

- keepalived.conf 파일에 아래 내용 제외 모두 삭제

```
[root@fa-test01 network-scripts]# cat /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP   # backup node는 MASTER -> BACKUP // nopreempt 모드기에 둘다 backup
    interface eth1   # vip를 secondary ip로 매핑한 인터페이스
    virtual_router_id 51 # VRRP 프로토콜로 advertisement 를 교환할 가상의 라우터
												# 동일한 id를 사용하는 virtual router가 있으면 오류 발생!
    priority 100 # 숫자가 높은 쪽이 Master가 되므로 backup은 100보다 낮게 설정
    advert_int 1  # 1초마다 advertisement 패킷 교환
    nopreempt   #Active 살아나도 절체 안함
    authentication {
        auth_type PASS # 패스워드 인증방식
        auth_pass 1111 # 비밀번호 1111
    }
    virtual_ipaddress {
        10.30.31.171 dev eth1 # VIP 주소와, 해당 VIP를 사용하여 통신할 인터페이스 지정
                              # dev eth1 부분은 생략해도 정상 동작
    }
}

```

### 4**. Backup Node에 Keepalived config 적용**

- keepalived.conf 파일에 아래 내용 제외 모두 삭제

```
[root@fa-test02 network-scripts]# cat /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER   # backup node는 MASTER -> BACKUP
    interface eth1   # vip를 secondary ip로 매핑한 인터페이스
    virtual_router_id 51 # VRRP 프로토콜로 advertisement 를 교환할 가상의 라우터
    priority 90 # 숫자가 높은 쪽이 Master가 되므로 backup은 100보다 낮게 설정
    advert_int 1  # 1초마다 advertisement 패킷 교환
    authentication {
        auth_type PASS # 패스워드 인증방식
        auth_pass 1111 # 비밀번호 1111
    }
    virtual_ipaddress {
        10.30.31.171 dev eth1 # VIP 주소와, 해당 VIP를 사용하여 통신할 인터페이스 지정
                              # dev eth1 부분은 생략해도 정상 동작
    }
}

```

### **7. network 및 keepalived 서비스 재시작**

- Master/Bkup 모두에서 수행

```
# systemctl restart network
# systemctl restart keepalived
# systemctl restart network

```

- 재부팅시 keepalived 서비스 자동시작 설정

```
$ systemctl enable keepalived

```

### **8. Failover 테스트 전 test용 vm에서 사전 준비**

- 통신 연속성 확인을 위한 ping 수행 (실행시켜놓은 상태 유지)

```
 # ping 10.30.31.171
```

- arp 테이블 확인을 통해 10.30.31.171 ip의 mac address 정보 변경 확인

```
[root@fa-test02 network-scripts]# arp -a
? (10.30.31.172) at 1e:00:a7:04:13:c3 [ether] on eth1
gateway (172.27.0.1) at 02:00:39:a3:07:8c [ether] on eth0
? (10.30.31.161) at 1e:00:79:04:13:b8 [ether] on eth1
r-109017-VM.cs7001cloud.internal (172.27.0.223) at 02:00:39:a3:07:8c [ether] on eth0
You have new mail in /var/spool/mail/root

```

### **9. Failover 테스트 전 backup node에서 사전 준비 사항**

- Bkup -> Master 상태 변경 확인을 위한 로그 정보 트래킹

```
# tailf /var/log/messages

```

### **10. Master Node에서 절체 수행**

- 인터페이스 다운 혹은 poweroff 수행

```
# poweroff
혹은
# ifconfig eth1 down
(업시킬땐 ifconfig eth1 up)

```

### **11. Backup Node에서 상태 변경 확인**

- 8번에서 수행해놓은 로깅을 통해 Backup -> Master 상태변경 확인

```
Jul 14 10:05:37 fa-test02 Keepalived_vrrp[15744]: VRRP_Instance(VI_1) Transition to MASTER STATE
Jul 14 10:05:38 fa-test02 Keepalived_vrrp[15744]: VRRP_Instance(VI_1) Entering MASTER STATE
Jul 14 10:05:38 fa-test02 Keepalived_vrrp[15744]: VRRP_Instance(VI_1) setting protocol VIPs.
Jul 14 10:05:38 fa-test02 Keepalived_vrrp[15744]: Sending gratuitous ARP on eth1 for 10.30.31.171
Jul 14 10:05:38 fa-test02 Keepalived_vrrp[15744]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on eth1 for 10.30.31.171
Jul 14 10:05:38 fa-test02 Keepalived_vrrp[15744]: Sending gratuitous ARP on eth1 for 10.30.31.171
Jul 14 10:05:40 fa-test02 ntpd[3858]: Listen normally on 16 eth1 10.30.31.171 UDP 123

```

- Test vm에서 ping 연속성 확인 및 ARP 테이블 확인

```
[root@fa-test03 ~]# arp -a
? (10.30.31.168) at 1e:00:c3:04:13:bf [ether] on eth1
? (10.30.31.161) at 1e:00:79:04:13:b8 [ether] on eth1
? (10.30.31.171) at 1e:00:c3:04:13:bf [ether] on eth1
gateway (172.27.0.1) at 02:00:39:a3:07:8c [ether] on eth0
? (10.30.31.170) at 1e:00:ec:04:13:c1 [ether] on eth1
r-109017-VM.cs7001cloud.internal (172.27.0.223) at 02:00:39:a3:07:8c [ether] on eth0
[root@fa-test03 ~]# arp -a
? (10.30.31.168) at 1e:00:c3:04:13:bf [ether] on eth1
? (10.30.31.161) at 1e:00:79:04:13:b8 [ether] on eth1
? (10.30.31.171) at 1e:00:ec:04:13:c1 [ether] on eth1   # ARP Table에서 VIP에 매핑된 MAC addr이 변경됨을
gateway (172.27.0.1) at 02:00:39:a3:07:8c [ether] on eth0
? (10.30.31.170) at 1e:00:ec:04:13:c1 [ether] on eth1
r-109017-VM.cs7001cloud.internal (172.27.0.223) at 02:00:39:a3:07:8c [ether] on eth0
```

# **DRBD (Distributed Replicated Block Device)**

- 특정한 하나의 볼륨을 다른 볼륨과 미러링
- 네트워크를 사용해 Raid 1 구조를 구현했다고 보면 됨
- Active / Standby 구조
- Active 다운시 자동으로 Standby로 넘어가지 않기 때문에, 별도 조작이 필요하다
    - FA에는 keepalived에 쉘스크립트를 추가하여 Standby<->Active 동작이 가능하도록 커스터마이징
- 동기화 프로토콜을 다양하게 제공하여 정합성 높음

## **DRBD 적용 순서**

1. Selinux disable
- cat /etc/sysconfig/selinux에서 SELINUX = disabled로 변경
    - selinux는 enable / permissive / disable 모드가 있음

```
[root@fa-test01 network-scripts]# cat /etc/sysconfig/selinux
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
```

1. 커널 최신버전으로 업데이트
- kmod-drbd90의 requirement로 인해 kernel update 필요
    - Requirements: [https://centos.pkgs.org/7/elrepo-x86_64/kmod-drbd90-9.0.22-2.el7_8.elrepo.x86_64.rpm.html](https://centos.pkgs.org/7/elrepo-x86_64/kmod-drbd90-9.0.22-2.el7_8.elrepo.x86_64.rpm.html)
- 현재 kernel version 확인

```
# uname -a
```

- 커널 업데이트 진행

```
# cd /root/pkgs/drbd
yum install kernel-ml<TAB>
-------------------------
상용 서버에서는 인터넷이 없으므로 위 내용이 아닌 rpm -ivh kernel-ml<TAB>으로 설치 수행
```

- 커널 부팅 순서 변경

```
# cat /boot/grub2/grub.cfg | grep menuentry | cut -d "'" -f2    #부팅순서확인
# grub2-set-default "CentOS Linux (5.4.12-1.el7.elrepo.x86_64) 7 (Core)"  #최신버전의 kernel을 default 설정
# grub2-editenv li
st  #부팅될 커널 확인
# reboot #재부팅
# uname -a #재부팅 후 로드된 커널 확인

```

1. drbd 패키지 설치

```
# yum -y install kmod-drbd90 drbd90-utils
```

1. 파티션 생성
- 서버에 physical disk가 인식만 되어있는 상황부터 시작
- ls -l /dev/XXX*, lsblk, fdisk-l, pvdisplay, lvdisplay 명령어 참고
- 테스트 vm 기준으로 /dev/xvdb에 파티션을 생성
- 파티션 생성 후 아래 순서대로 LV 생성

```
# fdisk /dev/xvdb -> primary, 파티션넘버, 전체 섹터로 생성 후 t를 눌러 시스템을 linux LVM으로 변경(n, p, 1, wq, t, 8e 순서대로 입력)
# pvcreate /dev/xvdb1                 -> /dev/xvdb의 파티션인 xvdb1에 pv 생성
# vgcreate drbdnode1 /dev/xvdb1           -> pv(/dev/xvdb1)에 vg을 drbd1의 이름으로 생성
# lvcreate -l 100%FREE -n drbdnode1lv drbdnode1 -> -l: 로지컬볼륨(PE), 9.9G, -n drbdnode1lv 이름으로, drbdnode1에 vg 생성
```

- 생성 후 display, lsblk로 조회
- 참고 : 파일시스템에서 사용하기 위해선 마운트해줘야함
- drbd는 primary(Active), secondary(backup)가 있는데, secondary에서는 미러링만 할 뿐 마운트되지 않는다.
- secondary를 마운트해주기 위해서는 primary를 secondary로, secondary를 primary로 변경해주어야 한다.
- drbd는 7788과 7799 port를 사용하므로, 해당 포트를 비워놔야함.
1. Configure resource
- drbd는 /etc/drbd.conf 파일로 컨트롤되나, 보통 해당 파일은 include "/etc/drbd.d/global_common.conf" 만 있으며
- /etc/drbd.d/global_common.conf 파일이나 /etc/drbd.d/*.res 파일을 생성해서 설정값을 조정한다.
- global_common.conf에서 protocol C 설정 후 개별 res 파일로 상세 설정을 한다.

```
# /etc/drbd.d/global_common.conf
global {
  usage-count yes;
}
common {
  net {
    protocol C;
  }
}
```

```
[root@fa-test01 ~]# cat /etc/drbd.d/r0.res
resource r0 {
        startup {
                wfc-timeout 30;
                degr-wfc-timeout 30;
        }
        net {
                cram-hmac-alg sha1;
                shared-secret sync_disk;
        }
        syncer {
                rate 100M;
                al-extents 257;
                on-no-data-accessible io-error;
        }
        on NODE1 {  #NODE1은 uname -a에서 확인되는 hostname이어야하며, /etc/hosts에도 동일한 이름으로 지정되어있어야한다.
                device /dev/drbd0;
                disk /dev/mapper/drbd1-drbd1lv;
                address 10.30.31.168:7788;
                meta-disk internal;
        }
        on NODE2 {  #NODE1은 uname -a에서 확인되는 hostname이어야하며, /etc/hosts에도 동일한 이름으로 지정되어있어야한다.
                device /dev/drbd0;
                disk /dev/mapper/drbd2-drbd2lv;
                address 10.30.31.170:7788;
                meta-disk internal;
        }
}
```

- drbd를 커널에 적재시킨다

```
# lsmod | grep drbd
# modprobe drbd
# lsmod | grep drbd
```

- drbd 메타데이터를 생성한다

```
drbdadm create-md r0 #r0: 리소스이름
```

1. 부팅시 자동 시작 설정

```
[root@fa-test02 ~]# systemctl is-enabled drbd
disabled
[root@fa-test02 ~]# systemctl enable drbd
Created symlink from /etc/systemd/system/multi-user.target.wants/drbd.service to /usr/lib/systemd/system/drbd.service.
[root@fa-test02 ~]# systemctl is-enabled drbd
enabled
```

1. drbd 구동
- Master에서 구동 후 완료되면 Backup에서도 구동한다.

```
[root@fa-test01 ~]# systemctl start drbd
[root@fa-test01 ~]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:WFConnection ro:Secondary/Unknown ds:Inconsistent/DUnknown C r----s
    ns:0 nr:0 dw:0 dr:0 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:10481308

[root@fa-test01 ~]# drbdadm status
r0 role:Secondary
  disk:Inconsistent
  peer role:Secondary
    replication:Established peer-disk:Inconsistent
```

**위 설정까지 master와 backup에서 모두 수행한다.**

**현재부터 Master에서만 설정**

1. Master 설정을 진행
- drbdadm primary --force 는 마스터 역할의 노드를 기준으로 다른 노드 쪽으로 복제 동기화를 수행하는 방법. 해당 방법은 초기에 복제해야하는 데이터의 양이 많지 않거나 시간적으로 여유가 있을 때 수행한다. 복제해야하는 양이 많으면 truck based replication을 수행해야한다 (참고: [https://atl.kr/dokuwiki/doku.php/drbd_%EC%82%AC%EC%9A%A9%EC%9E%90_%EC%95%88%EB%82%B4%EC%84%9C](https://atl.kr/dokuwiki/doku.php/drbd_%EC%82%AC%EC%9A%A9%EC%9E%90_%EC%95%88%EB%82%B4%EC%84%9C))

```
[root@fa-test01 ~]# drbdadm primary --force r0
[root@fa-test01 ~]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:SyncSource ro:Primary/Secondary ds:UpToDate/Inconsistent C r-----
    ns:434176 nr:0 dw:0 dr:436296 al:8 bm:0 lo:0 pe:156 ua:0 ap:0 ep:1 wo:f oos:10050460
        [>....................] sync'ed:  4.2% (9812/10232)M
        finish: 0:04:16 speed: 39,168 (39,168) K/sec
[root@fa-test01 ~]# drbdadm status
r0 role:Primary
  disk:UpToDate
  peer role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:21.52

```

- Resource r0을 Up 시켜준다

```
[root@fa-test01 ~]# drbdadm up r0
[root@fa-test01 ~]# drbdadm status
r0 role:Primary
  disk:UpToDate
  peer role:Secondary
    replication:Established peer-disk:UpToDate
[root@fa-test01 ~]# drbdadm status
r0 role:Primary
  disk:UpToDate
  peer role:Secondary
    replication:Established peer-disk:UpToDate
```

1. 파일시스템 생성 및 파티션 마운트
- 파일시스템 생성 전, drbd0이 LV에 disk로 생성되어있는지 확인

```
[root@fa-test01 ~]# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda                      202:0    0   20G  0 disk
├─xvda2                   202:2    0 19.1G  0 part
│ ├─centos-swap           253:1    0  1.9G  0 lvm  [SWAP]
│ └─centos-root           253:0    0 17.2G  0 lvm  /
└─xvda1                   202:1    0  953M  0 part /boot
sr0                        11:0    1 1024M  0 rom
xvdb                      202:16   0   10G  0 disk
└─xvdb1                   202:17   0   10G  0 part
  └─drbdnode1-drbdnode1lv 253:2    0   10G  0 lvm
    └─drbd0               147:0    0   10G  0 disk
```

- 파일시스템 생성 전 기존 사용 중인 파일시스템 포맷을 참고

```
[root@fa-test01 ~]# mount | grep ^/dev      # df -T 혹은 mount | grep ^/dev 로 확인 가능
/dev/mapper/centos-root on / type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/xvda1 on /boot type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

- 파일시스템 생성

```
[root@fa-test01 ~]# mkfs -t xfs /dev/drbd0
meta-data=/dev/drbd0             isize=512    agcount=4, agsize=655082 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620327, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

- 마운트할 디렉토리 생성

```
[root@fa-test01 ~]# mkdir /drbd-fs
[root@fa-test01 ~]# find / -name drbd-fs
/drbd-fs
[root@fa-test01 /]# df /drbd-fs
Filesystem              1K-blocks    Used Available Use% Mounted on
/dev/mapper/centos-root  18028544 2178684  15849860  13% /
```

- 마운트 수행

```
[root@fa-test01 /]# mount /dev/drbd0 /drbd-fs
[root@fa-test01 /]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 977M     0  977M   0% /dev
tmpfs                    991M     0  991M   0% /dev/shm
tmpfs                    991M  8.6M  982M   1% /run
tmpfs                    991M     0  991M   0% /sys/fs/cgroup
/dev/mapper/centos-root   18G  2.1G   16G  13% /
/dev/xvda1               950M  214M  736M  23% /boot
tmpfs                    199M     0  199M   0% /run/user/0
/dev/drbd0                10G   33M   10G   1% /drbd-fs
```

# **현재 단계로 구성 완료됐으며, 이후 Failover Test**

1. drbd filesystem에 테스트용 데이터 생성

```
[root@fa-test01 /]# vim /drbd-fs/testdata
[root@fa-test01 /]# ls /drbd-fs
testdata
```

1. Active, Standby node에서 drbd 상태 확인

```
#ACTIVE NODE
[root@fa-test01 /]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:10493946 nr:0 dw:12638 dr:10502422 al:15 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0

#STBY NODE
[root@fa-test02 ~]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:Connected ro:Secondary/Primary ds:UpToDate/UpToDate C r-----
    ns:0 nr:10493946 dw:10493946 dr:0 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
```

1. Failover Test를 위해 Master Node의 DRBD 서비스 stop

```
[root@fa-test01 /]# systemctl stop drbd
[root@fa-test01 /]# cat /proc/drbd
cat: /proc/drbd: No such file or directory
[root@fa-test01 /]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 977M     0  977M   0% /dev
tmpfs                    991M     0  991M   0% /dev/shm
tmpfs                    991M  8.6M  982M   1% /run
tmpfs                    991M     0  991M   0% /sys/fs/cgroup
/dev/mapper/centos-root   18G  2.1G   16G  13% /
/dev/xvda1               950M  214M  736M  23% /boot
tmpfs                    199M     0  199M   0% /run/user/0
[root@fa-test01 /]# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda                      202:0    0   20G  0 disk
├─xvda2                   202:2    0 19.1G  0 part
│ ├─centos-swap           253:1    0  1.9G  0 lvm  [SWAP]
│ └─centos-root           253:0    0 17.2G  0 lvm  /
└─xvda1                   202:1    0  953M  0 part /boot
sr0                        11:0    1 1024M  0 rom
xvdb                      202:16   0   10G  0 disk
└─xvdb1                   202:17   0   10G  0 part
  └─drbdnode1-drbdnode1lv 253:2    0   10G  0 lvm
```

1. Standby Node에서의 상태 확인
- 참고: DRBD는 자동으로 Active<->Standby 상태변환이 일어나지 않는다.

```
[root@fa-test02 ~]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:WFConnection ro:Secondary/Unknown ds:UpToDate/DUnknown C r-----
    ns:0 nr:10494004 dw:10494004 dr:0 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
[root@fa-test02 ~]# drbdadm status
r0 role:Secondary
  disk:UpToDate
  peer connection:Connecting
```

1. Standby 서버를 Primary 모드로 전환
- Primary로 전환 후 lsblk에서 drbd0 디스크가 잡히는 것을 확인할 수 있다.

```
[root@fa-test02 ~]# drbdadm primary r0
[root@fa-test02 ~]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:WFConnection ro:Primary/Unknown ds:UpToDate/DUnknown C r-----
    ns:0 nr:10494004 dw:10494004 dr:2120 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
[root@fa-test02 ~]# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda                      202:0    0   20G  0 disk
├─xvda2                   202:2    0 19.1G  0 part
│ ├─centos-swap           253:1    0  1.9G  0 lvm  [SWAP]
│ └─centos-root           253:0    0 17.2G  0 lvm  /
└─xvda1                   202:1    0  953M  0 part /boot
sr0                        11:0    1 1024M  0 rom
xvdb                      202:16   0   10G  0 disk
└─xvdb1                   202:17   0   10G  0 part
  └─drbdnode2-drbdnode2lv 253:2    0   10G  0 lvm
    └─drbd0               147:0    0   10G  0 disk
```

1. Standby 서버의 drbd 디스크 마운트
- 디스크를 마운트 한 후, 미러링된 정보도 확인한다.

```
[root@fa-test02 ~]# mkdir /drbd-fs
[root@fa-test02 ~]# mount /dev/drbd0 /drbd-fs
[root@fa-test02 ~]# ls /drbd-fs
testdata
```

- 테스트용 데이터를 추가 생성한다
1. Master 서버가 복구됐음을 가정하고 서비스 시작

```
[root@fa-test01 /]# systemctl start drbd
[root@fa-test01 /]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:Connected ro:Secondary/Primary ds:UpToDate/UpToDate C r-----
    ns:0 nr:2105 dw:2105 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
[root@fa-test01 /]# drbdadm status
r0 role:Secondary
  disk:UpToDate
  peer role:Primary
    replication:Established peer-disk:UpToDate
```

- /proc/drbd를 확인해보면 본래의 Master 서버가 Secondary 상태임을 확인할 수 있다.
1. Standby 서버를 secondary 모드로 전환

```
[root@fa-test02 ~]# umount /dev/drbd0
You have new mail in /var/spool/mail/root
[root@fa-test02 ~]# drbdadm secondary r0
[root@fa-test02 ~]# drbdadm status
r0 role:Secondary
  disk:UpToDate
  peer role:Secondary
    replication:Established peer-disk:UpToDate
```

1. Master 서버를 Primary 모드로 전환 및 데이터 생성 확인

```
[root@fa-test01 /]# drbdadm primary r0
[root@fa-test01 /]# cat /proc/drbd
version: 8.4.11 (api:1/proto:86-101)
srcversion: C83CE761848B9DE61379370
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:0 nr:2108 dw:2108 dr:2120 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
[root@fa-test01 /]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 977M     0  977M   0% /dev
tmpfs                    991M     0  991M   0% /dev/shm
tmpfs                    991M  8.6M  982M   1% /run
tmpfs                    991M     0  991M   0% /sys/fs/cgroup
/dev/mapper/centos-root   18G  2.1G   16G  13% /
/dev/xvda1               950M  214M  736M  23% /boot
tmpfs                    199M     0  199M   0% /run/user/0
[root@fa-test01 /]# mount /dev/drbd0 /drbd-fs
[root@fa-test01 /]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 977M     0  977M   0% /dev
tmpfs                    991M     0  991M   0% /dev/shm
tmpfs                    991M  8.6M  982M   1% /run
tmpfs                    991M     0  991M   0% /sys/fs/cgroup
/dev/mapper/centos-root   18G  2.1G   16G  13% /
/dev/xvda1               950M  214M  736M  23% /boot
tmpfs                    199M     0  199M   0% /run/user/0
/dev/drbd0                10G   33M   10G   1% /drbd-fs
[root@fa-test01 /]# ls /drbd-fs
fa-test02_made  testdata
```

# **현재 단계까지 DRBD 노드 구성이 완료됐으며, 다음 단계부터는 keepalived와 연계하여 Master Node 장애시 자동으로 Mount 설정**

1. /dev/drbd0을 Mount/Unmount해주는 쉘스크립트 작성
- /usr/local/sbin/

```
[root@fa-test01 sbin]# pwd
/usr/local/sbin

[root@fa-test01 sbin]# cat drbd_master.sh
#!/bin/sh
sudo drbdadm primary r0
sudo mkdir /drbd-fs
sudo mount /dev/drbd0 /drbd-fs

[root@fa-test01 sbin]# cat drbd_backup.sh
#!/bin/sh
sudo umount -f /dev/drbd0
sudo rmdir /drbd-fs
sudo drbdadm secondary r0
```

- 작성한 스크립트를 Backup 서버에도 복사

```
[root@fa-test01 sbin]# scp drbd_master.sh drbd_backup.sh root@10.30.31.170:/usr/local/sbin
The authenticity of host '10.30.31.170 (10.30.31.170)' can't be established.
ECDSA key fingerprint is SHA256:KHR7lHxKSdo3lCB2aX5K5mXX2lYilbGC4P3V+EDPZeM.
ECDSA key fingerprint is MD5:e2:d5:f5:f4:a2:40:04:70:c3:5a:5c:de:22:a3:60:ec.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.30.31.170' (ECDSA) to the list of known hosts.
root@10.30.31.170's password:
drbd_master.sh                                                                              100%   59    26.1KB/s   00:00
drbd_backup.sh                                                                              100%   49    19.6KB/s   00:00

# sudo chmod 755 ./*
```

1. keepalived의 Master/Backup 상태에 따라 스크립트가 호출되도록 설정
- /etc/keepalived/keepalived.conf 수정 (Master/Backup에서 모두 적용)

```
[root@fa-test01 sbin]# cat /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.30.31.171 dev eth1
    }
        notify_master "/usr/local/sbin/drbd_master.sh"   #추가된 부분
        notify_backup "/usr/local/sbin/drbd_backup.sh"   #추가된 부분
        notify_fault "/usr/local/sbin/drbd_backup.sh"    #추가된 부분
}
```

1. 파일 실행 권한 변경 (상용 서버의 유저 그룹 구성을 확인 후 바꿔야함)
- Master/backup 모두에서 수행

```
[root@fa-test02 ~]# chmod 755 /usr/local/sbin/drbd_backup.sh
[root@fa-test02 ~]# chmod 755 /usr/local/sbin/drbd_master.sh
```

1. keepalived 서비스 재시작

```
systemctl restart keepalived
```

- DRBD node dual down시 Split-brain 현상 발생
    - Split-Brain : 데이터가 싱크되는 중 외부 요인으로 인해 두 데이터가 다르게 되어 데이터 싱크 실패
        - 현상확인방법 : drbdadm connect all 명령어로도 양측에서 StandAlone인 상태로 나옴
    - 해결방법 : Secondary의 데이터를 모두 지우고 다시 싱크 실행

    ```jsx
    # Both node
    drbdadm disconnect all

    # Secondary node - 아래 명령어로 데이터 모두 삭제
    drbdadm --discard-my-data connect all

    # Both node - 아래 명령어를 통해 Secondary node의 Data를 모두 지우고 다시 Sync
    drbdadm connect all
    ```

drbd 절체 잘 안되면 스크립트 경로 확인 및 스크립트 내 sudo 추가(drbd_master/bkup.sh)

**keepalived + DRBD 서비스 구성 완료**

참고

- 한두개만 보지말고 여러곳을 보고 취합해서 공부해야함
- DRBD Manual : [https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/#p-build-install-configure](https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/#p-build-install-configure)
- 설명 잘된곳 : [https://atl.kr/dokuwiki/doku.php/drbd_%EC%82%AC%EC%9A%A9%EC%9E%90_%EC%95%88%EB%82%B4%EC%84%9C](https://atl.kr/dokuwiki/doku.php/drbd_%EC%82%AC%EC%9A%A9%EC%9E%90_%EC%95%88%EB%82%B4%EC%84%9C)
- 그나마 설명 좀 된곳 : [https://jirak.net/wp/centos7rhel7-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-drbd-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0/](https://jirak.net/wp/centos7rhel7-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-drbd-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0/)
- keepalived + drbd 구성 참고 : [http://coffeenix.net/data/files/20100317_infra_chapter3_by_jjun.pdf](http://coffeenix.net/data/files/20100317_infra_chapter3_by_jjun.pdf)
- [https://m.blog.naver.com/sjw9485/221448396266](https://m.blog.naver.com/sjw9485/221448396266)
- file:///D:/Download/%EC%8B%A4%EC%8A%B5%EA%B0%80%EC%9D%B4%EB%93%9C_DRBD%20(1).pdf
- [https://ddoriya.tistory.com/entry/DRBD-%EC%84%A4%EC%B9%98](https://ddoriya.tistory.com/entry/DRBD-%EC%84%A4%EC%B9%98)