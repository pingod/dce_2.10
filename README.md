## 部署前 ##
1. 己安装好centos 7.3/7.4操作系统
2. 准备ansible环境  
说明: ansible和离线源需要一台额外的主机, 安装完成后即可回收主机

``` shell
# 安装pip
yum -y install python-setuptools
easy_install pip
# 安装ansible
pip install ansible
```

3. 下载dce_2.10 ansible playbook
``` shell
git clone https://github.com/juneau-work/dce_2.10
cd dce_2.10
```
> 以下操作都以dce_2.10为basedir

4. 定义变量  

**dev/group_vars/vault**
> **注意:** 修改远程用户名密码及dce认证用户名密码与实际环境匹配
``` shell
cat <<'EOF' > vault.sh
VAULT_ID='myVAULT@2018'
echo $VAULT_ID > ~/.vault_pass.txt

ANSIBLE_USER='root' # ssh用户名
ANSIBLE_PASSWORD='root' # ssh用户密码
DCE_USER='admin' # 具有admin权限的dce认证用户
DCE_PASSWORD='admin' # 具有admin权限的dce认证用户密码

ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $ANSIBLE_USER --name 'vault_ansible_user' | tee dev/group_vars/vault
ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $ANSIBLE_PASSWORD --name 'vault_ansible_password' | tee -a dev/group_vars/vault
ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $DCE_USER --name 'vault_dce_user' | tee -a dev/group_vars/vault
ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $DCE_PASSWORD --name 'vault_dce_password' | tee -a dev/group_vars/vault
EOF
```
> 执行脚本
``` shell
bash vault.sh
```
**提示**: 后期新增加集群节点时，需要更新用户认证信息

**dev/group_vars/all**
> 需要修改的主要变量,其它变量请安需修改
``` yaml
# 组成thinpool的磁盘列表,多块磁盘用','分隔,如/dev/sdb,/dev/sdc
thinpool_disks: /dev/sdb 
# dce版本
dce_version: 2.10.1
# dce离线yum源
dce_offline_repo: http://192.168.130.1:15000/repo/centos-7.3.1611 
# dce离线镜像
dce_hub_prefix: 192.168.130.1:15000/daocloud
```

5. 配置主机列表**dev/hosts**  
	- seed是集群的第一台manager节点,用来初始化集群,只能是一个ip地址
	- manager是manager节点组
	- worker是worker节点组
``` ini
# 用熟悉的编辑器打找dev/hosts文件，如vim dev/hosts

[seed]
192.168.130.11

[manager]
192.168.130.12
192.168.130.12

[worker]
192.168.130.14
192.168.130.15
```



-------------------------------------------------------------------------------
- ### 离线安装(内网拉镜像) ###

#### 1. 搭建离线源 ####
> i. 下载离线安装包**dce-2.10.x.tar**  

[准备离线安装源](http://guide.daocloud.io/dce-v2.10/离线安装控制节点-13871615.html)  

> ii. 将离线包上传到一台安装有docker环境的主机，如: 192.168.130.1

``` shell
scp dce-2.10.1.tar root@192.168.130.1:/tmp
```

> iii. 启用离线源
``` shell
ssh root@192.168.130.1

export DCE_VERSION=2.10.1 # 修改为要安装的dce版本
export OS_VERSION=7.4.1708 # 修改为与操作系统匹配的版本
tar -xvf /tmp/dce-$DCE_VERSION.tar -C /tmp

cat > /etc/yum.repos.d/dce.repo <<EOF
[dce]
name=dce
baseurl=file:///tmp/dce-$DCE_VERSION/repo/centos-$OS_VERSION
gpgcheck=0
enabled=1
EOF
yum -y --disablerepo=\* --enablerepo=dce install docker-ce
systemctl start docker
rm -rf /etc/yum.repos.d/dce.repo

# 以容器方式运行registry, 默认端口为15000, 既提供dce离线镜像也提供docker, k8s等依赖包
cd /tmp/dce-$DCE_VERSION
./dce-installer up-installer-registry --image-path=dce-installer-registry.tar
```
![离线源安装成功](picture/001.jpg)
> iv. 测试离线源可用性

访问http://192.168.130.1:15000, 能成功看到目录索引则离线源配置成功

> v. 设置离线环境变量

dev/group_vars/all  
注意: **离线**安装请一定正确配置变量**dce_offline_repo, dce_hub_prefix**, 详见上文, 如果前面已经配置妥当，可以继续往下

#### 2. dce_installer ####
> 对应dce-installer prepare-docker
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install dce_installer.yml 
```
#### 3. init cluster ####
> 对应bash -c "$(docker run -i --rm -e HUB_PREFIX=<this-machine-ip>:15000/daocloud <this-machine-ip>:15000/daocloud/dce:2.10.0 install)"
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install seed.yml 
```
#### 4. join cluster ####
> 对应sudo bash -c "$(sudo docker run -i --rm -e HUB_PREFIX=<some-machine-ip>:15000/daocloud <some-machine-ip>:15000/daocloud/dce:2.10.0 join --token <Cluster-Token> <some-machine-ip>:2377 80)"
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install manager_or_worker.yml 
```





-------------------------------------------------------------------------------
- ### 在线安装(公网拉镜像) ###
> 在线安装相比离线安装少了第一步(准备离线源), 其它步骤完全一样  

注意: **在线**安装请一定**注释或删除**变量**dce_offline_repo, dce_hub_prefix**




-------------------------------------------------------------------------------
- ### 添加节点到己存在集群 ###
#### 1. dce_installer ####
> 只需在worker章节指定新worker节点ip, seed, manager都保持为空
``` ini
[seed]

[manager]

[worker]
192.168.130.100
192.168.130.101
```
> **注意:** 新主机如果主机名己经设置好，请一定通过--skip-tags hostname,hosts跳过主机名和hosts修改

``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install dce_installer.yml --skip-tags hostname,hosts
```

#### 2. join cluster ####
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install manager_or_worker.yml 
```




-------------------------------------------------------------------------------
- ### 卸载 ###
> **注意:** 卸载需谨慎, 请小心操作。将install置为uninstall,如
#### 1. 卸载manager或worker节点 ####
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=uninstall manager_or_worker.yml 
```
#### 2. 卸载seed节点 ####
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=uninstall seed.yml 
```
#### 3. 卸载dce依赖包(docker, k8s) ####
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=uninstall dce_installer.yml 
```





-------------------------------------------------------------------------------
- ### 离线升级 ###
> **注意:** 使用待升级的离线源版本，离线源配置同上
#### 1. 定义变量 ####
- dev/group_vars/all  
``` yaml
dce_version: 2.10.1
dce_hub_prefix: 192.168.130.1:15000/daocloud
```
- dev/hosts
``` ini
# 只需要在seed章节指定一台manager, 这台manager可以是非leader节点
[seed]
192.168.130.11
```
#### 2. pull新版dce镜像(所有manager, worker节点) ####
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt sync_image.yml
```
#### 3. 升级 ####
> **注意:** 只需升级manager节点中的任意一台即可，待升级完成后，其它manager节点和worker节点会自动升级到对应版本
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt upgrade.yml
```

