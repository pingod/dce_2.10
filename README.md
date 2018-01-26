### 环境变量: dev/group_vars/all ###
### 主机列表: dev/hosts ###

- ### 安装 ###
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

- ### 卸载 ###
> 将变量action置为uninstall,如
```
ansible-playbook -i dev/hosts --extra-vars install_or_uninstall=uninstall manager_or_worker.yml 
```
