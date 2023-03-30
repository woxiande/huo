# mysql 数据库相关

## 登录

账号：root 密码 root

```powershell
mysql -uroot -proot
```

## 创建数据库

创建名字为 test_db 数据库

```mysql
> CREATE DATABASE test_db;
```

## 删除数据库

```mysql
> DROP DATABASE test_db;
```

## 查询数据库

列出 mysql 中所有数据库

```mysql
SHOW DATABASES;
```

## 数据库导出

### 导出单个数据库

导出名字为 mydb 的数据库至

```powershell
mysqldump -uroot -proot mydb > /var/lib/mysql/mydb.sql
```

### 导出所有数据库 db 结构和数据

```powershell
mysqldump -uroot -proot -A > /var/lib/mysql/all.sql
```

## 数据库导入

### 导入单个数据库

系统命令行

```powershell
mysql -uroot -proot test_data < /var/lib/mysql/test_data.sql #test_data 名字未必一至，存在表格即可
```

mysql 命令行

```mysql
> use test_data; #选择数据库
> source /var/lib/mysql/test_data.sql; #将mysql/test_data.sql文件中数据导入所选择的数据库
```

### 导入所有数据库

mysql 命令行

```mysql
>source /var/lib/mysql/all.sql;
```

系统命令行

```powershell
mysql -uroot -proot < /var/lib/mysql/all.sql
```

# 问题处理

报错 1273 ：**ERROR 1273 (HY000): Unknown collation: 'utf8mb4_0900_ai_ci’**

处理方法：产生这种原因主要是因为 MySQL 版本不兼容导致的，要解决这个问题很简单，只需要打开 sql 文件，然后进行以下操作：

- 把文件中的所有的 utf8mb4_0900_ai_ci 替换为 utf8_general_ci
- 把文件中的所有的 utf8mb4 替换为 utf8
