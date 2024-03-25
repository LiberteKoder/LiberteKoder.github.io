### Receiving from client 是什么意思

The server is reading a packet from the network. 翻译过来的意思是：服务器正在从网络中读取数据包。

### 实验环境
1. Linux 内核版本5.10.134-16.1.al8.x86_64
2. MySQL8
3. sysvebch 版本：sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3

### 搭建实验环境
1. 首先我们在docker中安装一个MySQL：docker pull mysql 
2. 启动MySQL容器：docker run -it -d --net=host -e MYSQL_ROOT_PASSWORD=123 --name=mysql mysql
3. 进入MySQL容器中：docker exec -u root -it mysql bash
4. 登录MySQL：mysql -uroot -p ，输入密码
5. 创建测试数据库：create databses  test;
6. 退出MySQL bash ，随机生成数据：
`sysbench --mysql-user='root' --mysql-password='123' --mysql-db='test' --mysql-host='127.0.0.1' --mysql-port='3306' --tables='16'  --table-size='10000' --range-size='5' --db-ps-mode='disable' --skip-trx='on' --mysql-ignore-errors='all' --time='1180' --report-interval='1' --histogram='on' --threads=1 oltp_read_only prepare`
运行完毕上面的命令，就会生成一些测试的数据，接下来就可以进行正式的实验了。

### 实验步骤
1. 使用sysbench工具启动一个线程持续进行1180秒的只读性能测试。脚本如下：
2. `sysbench --mysql-user='root' --mysql-password='123' --mysql-db='test' --mysql-host='127.0.0.1' --mysql-port='3306' --tables='16' --table-size='10000' --range-size='5' --db-ps-mode='disable' --skip-trx='on' --mysql-ignore-errors='all' --time='1180' --report-interval='1' --histogram='on' --threads=1 oltp_read_only run `
3. 运行这个脚本之后，我们返回到MySQL的bash界面：执行 show processlist;查看MySQL服务端所有的链接和执行中的进程。返回结果如下：
![image](https://github.com/LiberteKoder/LiberteKoder.github.io/assets/133127435/21ca62e9-ea25-4500-b510-d27c46840661)
我们运行 kill pid 杀掉这个进程，再次运行 show processlist 怎会复现Receiving from client 错误，如下图：
![image](https://github.com/LiberteKoder/LiberteKoder.github.io/assets/133127435/30bb45e9-1016-4322-bec0-2f59a66288b7)


我自己的思路：
首先使用了netstat  查看网络链接状态：
        tcp       28      0 localhost:36394         localhost:mysql         CLOSE_WAIT 
	tcp       28      0 localhost:36394         localhost:mysql         CLOSE_WAIT 
	tcp       28      0 localhost:36394         localhost:mysql         CLOSE_WAIT
有大量的CLOSE_WAIT 状态。这表示msyql服务端已经发了关闭链接的信号，但是本地客户端还没有close。因为没有指定 -n 参数，netstat 一直在输出上面的结果，那么说明客户端一直在尝试链接。

那么到这我首先想到的是使用top命令，查看有没有一直在运行的进程：
![image](https://github.com/LiberteKoder/LiberteKoder.github.io/assets/133127435/c7f3417d-2654-4ce4-a83b-0bf8ea4c5e96)

我发现了 sysbench 进程占用CPU 高达85%，这就确定了是客户端一直在尝试链接MySQL。

其实这个实验按照我自己的想法已经结束了。但是群里有的小伙伴思考的比我深入的多。我也学到了好多有用的命令，在这记录一下。
perf top -p pid  :实时显示系统中最占用CPU的函数调用
strace -f -p 8601：对进程ID为8601的进程的系统调用跟踪，-f表示跟踪由该进程创建的所有子进程
sudo tshark -i lo -f "tcp port 3306" -c 20 ：抓MySQL端口3306的前20个包，本地通信(不管用哪个IP)都是走lo 网卡）**敲重点**
 
所有实验信息都在[程序员案例分享星球中](https://t.zsxq.com/18CjEcmmr)，我的思考深度是最浅的，星球中有很多大佬的奇技淫巧。
等任总直播完 再来更新。
