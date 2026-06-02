# 字符编码
## UTF-8
UTF-8（8位元，Universal Character Set/Unicode Transformation Format）是针对Unicode的一种可变长度字符编码。它可以用来表示Unicode标准中的任何字符，而且其编码中的第一个字节仍与ASCII相容，使得原来处理ASCII字符的软件无须或只进行少部分修改后，便可继续使用。因此，它逐渐成为电子邮件、网页及其他存储或传送文字的应用中，优先采用的编码。（[更多...](更多...)）

## GBK
GBK 是国家标准GB2312基础上扩容后兼容GB2312的标准。
GBK的文字编码是用双字节来表示的，即不论中、英文字符均使用双字节来表示，为了区分中文，将其最高位都设定成1。
GBK包含全部中文字符，是国家编码，通用性比UTF8差，不过UTF8占用的数据库比GBK大。

## utf8mb4
MySQL 5.5 之前，UTF8 编码只支持1-3个字节，只支持BMP这部分的unicode编码区。从mysql 5.5 开始，可支持4个字节UTF编码utf8mb4，一个字符最多能有4字节，所以能支持更多的字符集。

# 汉字长度与编码有关
5.0版本以下不考虑

UTF-8：一个汉字 = 3个字节，英文是一个字节
GBK： 一个汉字 = 2个字节，英文是一个字节

varchar(n) 表示n个字符，无论汉字和英文，MySql都能存入 n 个字符，仅实际字节长度有所区别。

MySQL检查长度，可用SQL语言
```sql
SELECT LENGTH(fieldname) FROM tablename 
```
# 验证
## utf8mb4 测试
```sql
SELECT ta.account_name ,LENGTH(ta.account_name) from t_account ta;
```
![image.png](https://github.com/oh-huohou/huohou.blog/blob/main/image/image-ccd9a176051a4eadafd88daf6fb93b83.png)
一个汉字占3个字节，一个字母占一个字节，一个表情占4个字节

##  GBK 测试
![image.png](https://github.com/oh-huohou/huohou.blog/blob/main/image/image-37addefd7c614d7594c2e501a2fb358a.png)
首先 插入表情失败
再来看汉字长度
```
SELECT account_name ,LENGTH(account_name) from t_temp;
```
![image.png](https://github.com/oh-huohou/huohou.blog/blob/main/image/image-9db01dd6f6364a73929504358600c446.png)


# varchar 最多能存多少值
mysql的记录行长度是有限制的，不是无限长的，这个长度是64K，即65535个字节，对所有的表都是一样的。

MySQL对于变长类型的字段会有1-2个字节来保存字符长度。
* 当字符数小于等于255时，MySQL只用1个字节来记录，因为2的8次方减1只能存到255。
* 当字符数多余255时，就得用2个字节来存长度了。

在utf-8状态下的varchar，最大只能到 (65535 - 2) / 3 = 21844 余 1。

在gbk状态下的varchar, 最大只能到 (65535 - 2) / 2 = 32766 余 1

在utf8mb4状态下的varchar，最大长度类似于utf8，取决于是否有4字节字符

## 使用 utf-8 创建
```sql
    CREATE TABLE `str_test` (
     `id`  tinyint(1)  NOT NULL,
     `name_chn` varchar(21845) NOT NULL
    ) ENGINE=InnoDB AUTO_INCREMENT=62974 DEFAULT CHARSET=utf8;
```
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs

把21845改为21844后Query OK, 0 rows affected (0.06 sec)

## 使用gbk创建
![image.png](https://github.com/oh-huohou/huohou.blog/blob/main/image/image-05dce524c336491da640c5265211bed4.png)
字节长度32765*2+存储长度使用字节长度2+int类型字节长度4=65536>65535
当存储长度为 32764 成功Query OK, 0 rows affected (0.03 sec)

参考：
https://www.cnblogs.com/printN/p/7252490.html
https://www.shuzhiduo.com/A/MAzA8kjpd9/