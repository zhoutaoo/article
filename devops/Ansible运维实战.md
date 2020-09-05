```shell
# 指定host运行
ansible-playbook site.yml -v -l hostIP或别名
# 指定远程用户运行
ansible-playbook site.yml -v -u 远程用户名
# 指定tag运行
ansible-playbook site.yml --tags "tag名"
# 跳过指定tag运行
ansible-playbook site.yml --skip-tags "tag名"

# 检查是否有变化
ansible-playbook site.yml --check/-C
# 检查语法，不运行
ansible-playbook site.yml --syntax-check

# 列出所有hosts
ansible-playbook site.yml --list-hosts
# 列出所有tag
ansible-playbook site.yml --list-tags/-t
# 列出所有task
ansible-playbook site.yml --list-tasks
# 指定inventory文件运行
ansible-playbook site.yml -v -i inventory文件
# 指定task位置开始运行
ansible-playbook site.yml --v -l hostIP或别名 --start-at="task name" 
```

