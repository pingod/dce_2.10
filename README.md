### 环境变量: dev/group_vars/all ###
### 主机列表: dev/hosts ###

- ### 在线安装 ###
> 定义变量dce_hub_prefix: daocloud.io/daocloud 或 删除变量dce_hub_prefix
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

- ### 离线安装 ###
> [准备离线安装源](http://guide.daocloud.io/dce-v2.10/离线安装控制节点-13871615.html)
定义变量dce_hub_prefix: 192.168.130.1:15000/daocloud

#### 1. common ####
```
ansible-playbook -i dev/hosts common.yml 
```
#### 2. dce_installer ####
> --extra-vars offline=true
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install --extra-vars offline=true dce_installer.yml 
```
#### 3. init cluster ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install seed.yml 
```
#### 4. join cluster ####
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=install manager_or_worker.yml 
```

- ### 卸载 ###
> 将变量action置为uninstall,如
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=uninstall manager_or_worker.yml 
```
