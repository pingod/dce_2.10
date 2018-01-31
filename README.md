## 部署前准备 ##
- 己安装好centos 7.4操作系统
- 配置主机列表**dev/hosts**
- 定义变量**dev/group_vars/all**

-------------------------------------------------------------------------------

- ### 在线安装 ###
#### 1. common ####
```
ansible-playbook -i dev/hosts common.yml 
```
#### 2. dce_installer ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install dce_installer.yml 
```
#### 3. init cluster ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install seed.yml 
```
#### 4. join cluster ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install manager_or_worker.yml 
```

-------------------------------------------------------------------------------

- ### 离线安装 ###

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
``` yaml
dce_offline_repo: http://192.168.130.1:15000/repo/centos-7.4.1708
dce_hub_prefix: 192.168.130.1:15000/daocloud  
```

#### 2. common ####
```
ansible-playbook -i dev/hosts common.yml 
```
#### 3. dce_installer ####
> **--extra-vars offline=true**
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install --extra-vars offline=true dce_installer.yml 
```
#### 4. init cluster ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install seed.yml 
```
#### 5. join cluster ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install manager_or_worker.yml 
```

-------------------------------------------------------------------------------

- ### 卸载 ###
> **注意:** 卸载需谨慎，请小心操作。
> 将install置为uninstall,如
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=uninstall manager_or_worker.yml 
```
