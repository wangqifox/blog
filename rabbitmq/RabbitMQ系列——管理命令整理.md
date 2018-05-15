---
title: RabbitMQ系列——管理命令整理
date: 2018/05/09 15:39:00
---

- 查看当前所有用户

    `rabbitmqctl list_users`
    
- 删除用户

    `rabbitmqctl delete_user username`
    <!-- more -->
- 添加新用户

    `rabbitmqctl add_user username password`

- 设置用户的tag

    `rabbitmqctl set_user_tags username administrator`

- 赋予用户默认vhost的全部操作权限

    `rabbitmqctl set_permissions -p / username ".*" ".*" ".*"`

- 查看用户的权限

    `rabbitmqctl list_user_permissions username`

- 开启web管理页面

    `rabbitmq-plugins enable rabbitmq_management`
    
    访问`http://127.0.0.1:15672/`地址可以进入web管理页面

`rabbitmqctl`命令的参考手册: [https://www.rabbitmq.com/rabbitmqctl.8.html](https://www.rabbitmq.com/rabbitmqctl.8.html)


