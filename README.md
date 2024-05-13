# Minio ansible role
使用Ansible安装minio服务
## 功能支持
1. 安装minio单机/集群模式
2. 批量创建Bucket、设置文件生命周期
3. 批量创建Bucket同步服务
4. 创建用户
5. 创建策略
6. 关联用户策略

## 定义主机清单
主机清单文件`minio_hosts`
```
minio_hosts:
  hosts:
    minio01:
      ansible_host: 20.1.1.101
    minio02:
      ansible_host: 20.1.1.102
    minio03:
      ansible_host: 20.1.1.103
```
## 自定义Playbook
请参考`roles/ansible-minio/defaults/main.yml`中的默认变量将自定义部分添加在自定义playbook，或者主机清单中
```
---
- name: Install minio service
  hosts: minio_hosts
  gather_facts: false
  vars:
    cluster_mode: true
    minio_basedir: /ngcs/minio
    minio_admin_user: admin
    minio_admin_pass: admin123
    # 新增用户设置
    users:
      - username: miniouser
        password: P@ssword
    # Bucket 列表
    bucket_list:
      - name: bucket01
        expire_days: 365 # 设置文件到期时间
      - name: bucket02
      - name: bucket03
      - name: bucket04
        expire_days: 180
    # Bucket同步服务
    sync_service:
      - upload_limit: 10MiB # 同步速率限制
        from:
          area_label: sh # 本地minio站点地域标识
        to:
          area_label: wx # 远程minio站点地域标识
          host: 127.0.0.1 # 远程minio服务IP地址
          port: 9000 # 远程minio服务端口号
          access_key: admin # 远程minio认证访问key（可使用为minio服务创建的默认管理员用户）
          secret_key: admin123 # 远程minio密钥（可使用为minio服务创建的默认管理员密码）
        bucket_mapping: # 同步的Bucket列表，local为本地的Bucket name，remote为远程的Bucket name
          - local: bucket01
            remote: bucket03
          - local: bucket02
            remote: bucket04
  roles:
    - ansible-minio
```
## 执行ansible命令
### 1. 安装Minio
#### 安装集群模式
集群模式会自动将
```
ansible-playbook -i minio_hosts minio.yml --tags install
```
#### 单机模式安装
安装单机模式，在playbook中添加变量`cluster_mode: false`
```
ansible-playbook -i minio_hosts minio.yml --tags install
```
### 2. 创建Bucket
请将需要创建的Bucket，添加bucket_list这个列表中，参考ansible-minio/defaults/main.yml
```
...
bucket_list:
  - bucket1
  - bucket2
```
执行创建命令
```
ansible-playbook -i minio_hosts minio.yml --tags create_buckets
```
### 3. 创建Bucket同步服务
创建同步服务时，请注意在变量中添加sync_service部分配置，参考ansible-minio/defaults/main.yml
```
ansible-playbook -i minio_hosts minio.yml --tags create_sync_services
```
### 4. 创建用户，并赋予对应的权限
默认会给用户读写的权限，详见`files/getput_policy.json`，如果需要新增一些其他的权限，可以通过下列步骤实现

- 创建json格式的权限配置，放入files目录中，比如read.json
- 编辑playbook，设置变量
  ```
  users:
    - username: miniouser
      password: P@ssword
      policies: # 设置给这个用户关联的权限
        - name: read
          file: read.json
  ```
```
ansible-playbook -i sit/minio_hosts minio.yml --tags install,create_users
```

### 5. 安装、创建Bucket、创建Bucket同步服务、创建用户
推荐将安装Minio、创建Bucket和创建Bucket同步服务分开执行，如果确认自己的设置没有问题，也可以一条命令完成。
```
ansible-playbook -i minio_hosts minio.yml --tags install,create_buckets,create_sync_services,create_users
```
### 6. 卸载Minio
这个操作不可恢复，请务必慎重
```
ansible-playbook -i minio_hosts minio.yml --tags uninstall
```
## 常用管理命令
启动minio服务
```
ansible-playbook -i minio_hosts minio.yml --tags start
```
停止minio服务
```
ansible-playbook -i minio_hosts minio.yml --tags stop
```
重启minio服务
```
ansible-playbook -i minio_hosts minio.yml --tags restart
```
