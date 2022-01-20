# kolla-ansible
Kolla-ansible을 사용해서 OpenStack을 구축한다. Kolla-ansible은 기본적으로 Docker 및 ansible를 사용해 Ansible과 도커에 대한 사전지식이 있으면 좋다.

 

Kolla-ansible 이란?
Kolla-ansible은 OpenStack 클라우드 운영을 위한 Docker 컨테이너 및 ansible 플레이북을 제공해준다.

기본적으로 매우 독립적이며 사용자의 요구에 따라 커스터 마이징이 가능하다.
```
환경셋팅
- HOST -
	- OS: Ubntu 20.04
	- MEM: 16G
	- Hypervisor: QEMU/KVM
	- VM: Cetnos 8

- GUEST -

- Controller Node: 
    - MEM: 8192MB
    - CPU: 4core
    - Disk: 50G
    - NIC: 
    	- NAT: 192.168.100.10/24
        - Internal: 192.168.110.10/24
    	- Hostname: controller

- Compute Node: 
	- MEM: 8192MB
    - CPU: 4core
    - Disk: 50G
    - NIC:
    	- NAT: 192.168.100.20/24
        - Internal: 192.168.110.20/24
        - Stroage(Internal): 192.168.120.20/24
    - Hostname: compute1

- Network Node:
	- MEM: 2048
    - CPU: 2core
    - Disk: 20G
    - NIC:
    	- NAT: 192.168.100.30/24
        - NAT(DHCP)
    - Hostname: network
  
 - Storage Node:
 	- MEM: 2048
    - CPU: 2core
    - Disk: 20G 2개
    - NIC:
    	- NAT: 192.168.100.40/24
        - Internal: 192.168.110.40/24
        - Storage(Internal): 192.168.120.40/24
    - Hostname: Storage
```
# OpenStack Install
공식 문서를 참고하여 설치를 진행하며 이 글에서는 Victoria 버전으로 설치를 진행한다.

 

## 1. 사전셋팅
```
1) 환경셋팅에 나와있는 네트워크로 네트워크를 설정해준다.

2) 각 Node에 Hostname을 변경 해준다.

3) /etc/sudoers.d/user에 다음과 같이 sudo 권한을 부여해준다.
user ALL=(ALL) NOPASSWD: ALL
```
## 2. 종속성 설치
### 1) python bulid 종속성 설치
```
[user@controller~] $ sudo dnf -y install python3-devel libffi-devel gcc openssl-devel python3-libselinux
```
### 2) 가상환경 활성화
```
[user@controller~] $ python3 -m venv ~/kolla
[user@controller~] $ source ~/kolla/bin/activate
```
### 3) Ansible 설치 및 설정 파일 생성
```
(kolla) [user@controller~] $ pip install -U pip
(kolla) [user@controller~] $ pip install ‘ansible<3.0’
(kolla) [user@controller~] $ sudo mkdir  /etc/ansible
(kolla) [user@controller~] $ sudo vi /etc/ansible/ansible.cfg
(kolla) [user@controller~] $ cat /etc/ansible/ansible.cfg
[defaults] 
host_key_checking=False 
pipelining=True 
forks=100
```
### 4) SSH 키기반 설정 Controller키 배포
```
(kolla) [user@controller~] $ ssh-keygen 
(kolla) [user@controller~] $ ssh-copy-id $USER@localhost
(kolla) [user@controller~] $ ssh-copy-id $USER@192.168.100.10 
(kolla) [user@controller~] $ ssh-copy-id $USER@192.168.100.20
(kolla) [user@controller~] $ ssh-copy-id $USER@192.168.100.30
(kolla) [user@controller~] $ ssh-copy-id $USER@192.168.100.40
```
## 3. Kolla-ansible 설치 및 구성
### 1) Kolla-ansible 다운로드
```
(kolla) [user@controller~] $ pip install kolla-ansible
```
### 2) /etc/kolla 디렉토리 생성 및 소유자 변경
```
(kolla) [user@controller~] $ sudo mkdir -p /etc/kolla 
(kolla) [user@controller~] $ sudo chown $USER:$USER /etc/kolla
```
### 3) globals.yml,password.yaml 파일 복사
```
(kolla) [user@controller~] $ cp -r ~/kolla/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```
### 4) 인벤토리 파일 복사
```
(kolla) [user@controller~] $ cp ~/kolla/share/kolla-ansible/ansible/inventory/* /etc/kolla
```
## 4. 인벤토리 수정
```
(kolla) [user@controller~] $ vim /etc/kolla/multinode

[control]

controller		ansible_host=192.168.100.10      ansible_become=true

[compute]

compute1		ansible_host=192.168.100.20      ansible_become=true

[network]

network			ansible_host=192.168.100.30      ansible_become=true

[storage]

storage			ansible_host=192.168.100.40      ansible_become=true

[monitoring]

[deployment]

localhost       ansible_connection=local
…….
```
## 5. 패스워드 파일 수정
```
(kolla) [user@controller~] $ kolla-genpwd
(kolla) [user@controller~] $ vim /etc/kolla/passwords.yml
(kolla) [user@controller~] $ cat /ett/kolla/passwords.yml
.....
keystone_admin_password: ******
.....
```


## 6. Globals.yml 변수 파일 수정
```
(kolla) [user@controller~] $ vi /etc/kolla/globals.yml

.......
kolla_base_distro: "centos" 
kolla_install_type: "binary" 
openstack_release: "victoria" 
kolla_internal_vip_address: "192.168.110.250" 
kolla_external_vip_address: "192.168.100.250" 
network_interface: "eth1" 
kolla_external_vip_interface: "eth0" 
neutron_external_interface: "eth2" 
enable_cinder: "yes" 
enable_cinder_backend_lvm: "yes" 
glance_backend_file: "yes"
nova_compute_virt_type: "qemu"
.......
```
## 7. Storage Node  Volume 준비
```
[user@storage~] $ sudo pvcreate /dev/vdb 
[user@storage~] $ sudo vgcreate cinder-volumes /dev/vdb
```
## 8. Openstack 배포
### 1) bootstrap
```
(kolla) [user@controller~] $ kolla-ansible -i /etc/kolla/multinode bootstrap-servers
```
### 2) prechecks
```
(kolla) [user@controller~] $ kolla-ansible -i /etc/kolla/multinode prechecks
```
### 3) image pull
```
(kolla) [user@controller~] $ kolla-ansible -i /etc/kolla/multinode pull
```
### 4)  deploy
```
(kolla) [user@controller~] $ kolla-ansible -i /etc/kolla/multinode deploy
```
## 9. OpenStack 대쉬보드 접속
kolla_external_vip_address 에서 설정한 IP 주소로 접속하면 생성된 오픈스택 화면을 볼 수 있다.

