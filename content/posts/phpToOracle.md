---

title: PHP使用PDO连接Oracle
date: 2018-10-16 17:00:00
tags: [PHP,Oracle]
---
# 前言
　　一直以来，Mysql和php可以说是标配，使用起来基本不需要额外的配置。但最近给学校做的一些项目里多数用到的是Oracle，初次尝试时还遇到了很多坎。这是第四次搭建php和Oracle环境了，前面踩了很多坑，这里就记录下。
  <!--more-->

###  环境

- Windows Server 2008 R2
- phpStudy 2018 (php 7.2.10)
- Oracle11g_home1

### 步骤
#####  安装phpStudy

本次在windows下搭建环境直接就使用 phpStudy了，但其有一些漏洞，后面在准备上线时再详细一说。

#####  修改php配置

在php-ini中将  `extension=php_oci8_12c.dll ` `extension=php_pdo_oci.dll` 两个扩展的注释去掉

![php-ini](https://blog-1252921857.cos.ap-chengdu.myqcloud.com/phppdo/1.jpg)

在PHP扩展中打开 `php_oci8` `php_oci8_11g` `php_pdo_oci`和其他项目中需要用到的扩展

重启phpStudy

##### 配置oracle客户端

在php中连接Oracle和连接Mysql不同，此时需要下载相应的Oracle的客户端，我们可以到[Oracle官网](https://www.oracle.com/technetwork/database/database-technologies/instant-client/downloads/index.html)进行下载。

此处需要格外注意的是客户端版本的选择，Oracle Instant Client的版本选择应该是要和**`PHP的版本`**相同而不是和**`操作系统`**的位数相同。此处使用的是phpStudy，其中的php是32位的，所以此处要选择32-bit的Oracle Instant Client进行下载。

然后添加两个系统变量并修改PATH变量

- `NLS_lANG` 保存为SIMPLIFIED CHINESE_CHINA.ZHS16GBK，为了解决读取编码问题
- `PATH`  添加客户端目录

> 此处为了使系统变量立即生效而不重启，可以进入CMD，输入 `set PATH=C:`然后重启CMD，再输入`echo %PATH%` ，此番便可以看到，系统变量已经生效。 

此时重启phpStudy，然后刷新phpinfo界面，再PDO drivers 处应该就可以看到oci被开启

![pdo](https://blog-1252921857.cos.ap-chengdu.myqcloud.com/phppdo/3.jpg)

> 如果此时该处无oci被开启的提示，可以在php的目录下执行`php -m`查看模块是否加载
>
> 如果是新装的机子，可能会出现`msvcr110.dll`,`msvcr100.dll`等链接库找不到,或者模块无法加载，此时下载相应版本的文件并放到正确的文件夹内即可。我这里出现的问题就是这一个，将动态库补充完整就会显示出来

##### PDO连接数据库

此处给出pdo连接的一个小例子:

```php
define('TNS', "(DESCRIPTION =(ADDRESS_LIST =(ADDRESS = (PROTOCOL = TCP)(HOST = 127.0.0.0.1)(PORT = 1521)))(CONNECT_DATA =(SERVICE_NAME = orcl)))");
        $db_username = "user";
        $db_password = "pass";
        try {
            $this->pdo = new PDO("oci:dbname=" . TNS . ';charset=utf8', $db_username, $db_password);
            $this->pdo->query("SET NAMES GBK");  // $_pdo->exec('SET NAMES utf8');  //设置数据库编码，两种方法都可以
        } catch (PDOException $e) {
            echo($e->getMessage());
        }
```

此处需要注意的是,在php Manual上，官方给出的例子是将`tns`定义为一个变量，而在我的环境的实际操作中，这样并不能连接成功，而将其定义为常量时，一切恢复正常

##### 加强phpStudy

以上操作结束后，就可以进行业务的开发了。但是如果作为正式上线的项目话，如果被漏扫，会被扫出很多漏洞。学校在我还没有开发完就给我扫了一遍，结果给我了一份长达95页的漏洞报告，而其中90%的都是目录浏览的漏洞，还有一些跨站脚本的问题。

因为phpStudy多作为开发环境，有很多利于开发者测试的配置默认开启，因此如果作为线上使用的话，建议进行配置更改。我这里做了如下操作:

- 删除www目录下phpMyadmin和index.php,l.php,phpinfo.php等危险脚本
- 设置目录列表禁止浏览
- 通过添加`server_tokens off`关闭nginx版本显示
- 修改Mysql默认密码,甚至删除
- 删除无用服务

---

此处再记录一个phpStudy因为nginx配置而引发的文件类型错误解析漏洞:

因为nginx的不当配置,使得像图片等类型的文件可以被当作php脚本被执行，详细可参考[这一篇文章](https://my.oschina.net/mark35/blog/33597)

另外还有当时遇到的另一个问题

> SQLSTATE[HY000]: OCIEnvNlsCreate: Check the character set is valid and that PHP has access to Oracle libraries and NLS data (ext\pdo_oci\oci_driver.c:619).

这个问题也是绞尽脑汁，各种搜索请教迟迟不能解决,在stackoverflow上提问也无济于事，后来自己蜜汁修好了... 解决方法在此:[Magic Problem](https://stackoverflow.com/questions/50202449/pdos-error-with-oracle)



