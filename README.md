## 部署前准备 ##
1. 己安装好centos 7.3/7.4操作系统
2. 准备ansible环境

``` shell
# 安装pip
yum -y install python-setuptools
easy_install pip
# 安装ansible
pip install ansible
```

3.  配置主机列表**dev/hosts**  
	- seed是种子节点,用来初始化集群,只能是一个ip地址
	- manager是manager节点组
	- worker是worker节点
4.  定义变量**dev/group_vars/all**
> **注意:** 对于敏感数据，如远程用户名密码及dce认证用户名密码, 请事先通过以下脚本生成密文
``` shell
VAULT_ID='myVAULT@2018'
echo $VAULT_ID > ~/.vault_pass.txt

ANSIBLE_USER='root'
ANSIBLE_PASSWORD='root'
DCE_USER='foo'
DCE_PASSWORD='bar'

ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $ANSIBLE_USER --name 'vault_ansible_user' | tee dev/group_vars/vault
ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $ANSIBLE_PASSWORD --name 'vault_ansible_password' | tee -a dev/group_vars/vault
ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $DCE_USER --name 'vault_dce_user' | tee -a dev/group_vars/vault
ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $DCE_PASSWORD --name 'vault_dce_password' | tee -a dev/group_vars/vault
```

-------------------------------------------------------------------------------


- ### 离线安装(内网拉镜像) ###

#### 1. 搭建离线源 ####
> i. 下载离线安装包**dce-2.10.0.tar**  

[准备离线安装源](http://guide.daocloud.io/dce-v2.10/离线安装控制节点-13871615.html)  

> ii. 将离线包上传到一台安装有docker环境的主机，如: 192.168.130.1

``` shell
scp dce-2.10.0.tar root@192.168.130.1:/tmp
```

> iii. 启用离线源

```shell
ssh root@192.168.130.1
tar -xvf /tmp/dce-2.10.0.tar -C /tmp
cd /tmp/dce-2.10.0
./dce-installer up-installer-registry --image-path=dce-installer-registry.tar
```
> iv. 测试离线源可用性

访问http://192.168.130.1:15000, 能成功看到目录索引则离线源配置成功

> v. 设置离线环境变量

dev/group_vas/all  
注意: **离线** 安装请一定正确配置变量 **dce_offline_repo, dce_hub_prefix**
``` yaml
dce_offline_repo: http://192.168.130.1:15000/repo/centos-7.4.1708
dce_hub_prefix: 192.168.130.1:15000/daocloud  
```

#### 2. dce_installer ####
```
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install dce_installer.yml 
```
#### 3. init cluster ####
```
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install seed.yml 
```
#### 4. join cluster ####
```
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install manager_or_worker.yml 
```

-------------------------------------------------------------------------------

- ### 在线安装(公网拉镜像) ###
> 在线安装相比离线安装少了第一步(准备离线源), 其它步骤完全一样  

注意: **在线**安装请一定 **注释或删除** 变量 **dce_offline_repo, dce_hub_prefix**

-------------------------------------------------------------------------------

- ### 卸载 ###
> **注意:** 卸载需谨慎, 请小心操作。将install置为uninstall,如
```
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=uninstall manager_or_worker.yml 
```
