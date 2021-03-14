# Windows下安装mysql


文档传送门(https://dev.mysql.com/doc/refman/8.0/en/windows-installation.html)

从noinstall ZIP Archive安装

初始化data目录
```
mysqld --initialize --console
```

登录, 输入随机生成的密码
```
mysql -u root -p
```

修改密码
```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'your-password';
```

