
Couchbase安装
==============
1. 安装操作系统对应couchbase服务，安装后，系统会在8091端口启动服务，使用浏览器访问本地的8091端口，会出现设置界面。在设置界面中，首先设置数据库的安装位置，如果是linux系统，需要将指定的目录设置为couchbase用户可以完全控制。
2. 设置系统使用的内存和服务其他设置：
3. hostname为服务绑定的ip地址，如果设置localhost，则只限于本机服务访问此数据库，如果外部系统要直接访问此数据库，则需设置具体的ip地址。
4. RAM quota为服务预留的内存，此时可以设置1G，用于分配给各个bucket使用，后期如果不够可以修改增加。其余设置可以默认
5. Sample bucket：可以不安装
6. default bucket：可以skip跳过
7. Notification：使用默认设置
8. Configure Server：需要设置密码，此密码在登录数据库的网页管理界面及执行后台命令时会使用

安装后设置：
1. 设置后进入服务界面，进入Data Buckets页面，使用Create Bucket 功能建立两个bucket：signalstudio和maptile。
2. signalstudio为基础服务数据库，存储主要的系统数据，设置时只需要修改名称和内存配置，最低500M即可
3. maptile为地图服务使用的数据bucket，100M内存即可
4. 建立bucket后可以使用cbrestore命令恢复之前备份的数据库文件：
    cbrestore 备份目录 http://localhost:8091 -b signalstudio -B signalstudio 
5. 备份数据库的方法：（备份的目的目录需要提前创建）
    cbbackup http://localhost:8091 备份目录 -b signalstudio

注：备份和恢复数据库的命令，在windows系统上的位置是数据库的安装目录Server/bin下，linux在/opt/couchbase/bin下。

couchbase具体文档请参照官方网址：

https://developer.couchbase.com/documentation/server/4.5/introduction/intro.html
