1. 用sed修改test.txt的23行test为tset；
```
   sed –i ‘23s/test/tset/g’ test.txt
```
2. 查看/web.log第25行第三列的内容。
```
   sed –n ‘25p’ /web.log | cut –d “ ” –f3
   head –n25 /web.log | tail –n1 | cut –d “ ” –f3
   awk –F “ ” ‘NR==23{print $3}’ /web.log
   ```
3. 删除每个临时文件的最初三行。
```
   sed –i ‘1,3d’ /tmp/*.tmp
   ```
4. 脚本编程：求100内的质数。
```
   #!/bin/bash
   i=1
   while[ $i -le 100 ];do
       ret=1
       for(( j=2;j<$i;j++ ));do
   if [ $(($i%$j))-eq 0  ];then
ret=0
break
   fi
       done
       if[ $ret -eq 1 ];then
           echo-n "$i "
       fi
       i=$((i+1 ))
   done
   ```
5. 晚上11点到早上8点之间每两个小时查看一次系统日期与时间，写出具体配置命令
```
   echo1 23,1-8/2 * * * root /tmp/walldate.sh >> /etc/crontab
   ```
6. 编写个shell脚本将当前目录下大于10K的文件转移到/tmp目录下
```
   #!/bin/bash
   fileinfo=($(du./*))
   length=${#fileinfo[@]}
   for((i=0;i<$length;i=$((i+2 ))));do
       if[ ${fileinfo[$i]} -le 10 ];then
   mv ${fileinfo[$((i+1 ))]} /tmp
       fi
   done
   ```
7. 如何将本地80端口的请求转发到8080端口，当前主机IP为192.168.2.1
```
   /sbin/iptables-t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.2.1:8080
   /sbin/iptables-t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to 8080
   ```
8. 在11月份内，每天的早上6点到12点中，每隔2小时执行一次/usr/bin/httpd.sh 怎么实现
```
   echo"1 6-12/2 * * * root /usr/bin/httpd.sh >> /etc/crontab"
   ```
9. 在shell环境如何杀死一个进程？
```
   psaux  | grep | cut -f? 得到pid
   cat/proc/pid
   killpid
   ```
10. 在shell环境如何查找一个文件？
```
   find/ -name abc.txt
   ```
11. 在shell里如何新建一个文件？
   touch~/newfile.txt
12. linux下面的sed和awk的编写
1）如何显示文本file.txt中第二大列大于56789的行？
```
  awk -F "," '{if($2>56789){print $0}}' file.txt
  ```
2）显示file.txt的1,3,5,7,10,15行？
```
  sed -n "1p;3p;5p;7p;10p;15p" file.txt
  awk 'NR==1||NR==3||NR==5||…||NR=15{print $0}' file.txt
  ```
3）将file.txt的制表符，即tab，全部替换成"|"
```
   sed-i "s#\t#\|#g" file.txt
   ```
13. 把当前目录（包含子目录）下所有后缀为“.sh”的文件后缀变更为“.shell”  
```
   #!/bin/bash
   str=`find./ -name \*.sh`
   fori in $str
   do
       mv$i ${i%sh}shell
   done
   ```
14. 编写shell实现自动删除50个账号功能，账号名为stud1至stud50
```
  #!/bin/bash
  for((i=1;i<=50;i++));do
      userdel stud$i
  done
  ```
15. 请用Iptables写出只允许10.1.8.179 访问本服务器的22端口。
```
  /sbin/iptables -A input -p tcp -dport 22 -s 10.1.8.179 -j ACCEPT
  /sbin/iptables -A input -p udp -dport 22 -s 10.1.8.179 -j ACCEPT
  /sbin/iptables -P input -j DROP
  ```
16. 在shell中变量的赋值有四种方法，其中，采用name=12的方法称（   A  ）。

A直接赋值                    B使用read命令
C使用命令行参数            D使用命令的输出

17. 有文件file1
1)查询file1里面空行的所在行号
```
  grep -n ^$ file1
  ```
2)查询file1以abc结尾的行
```
  grep abc$ file1
  ```
3)打印出file1文件第1到第三行
```
  head -n3 file1
  sed "3q" file1
  sed -n "1,3p" file1
  ```
18. 假设有一个脚本scan.sh，里面有1000行代码，并在vim模式下面，请按照如下要求写入对应的指令
1）将shutdown字符串全部替换成reboot
```
  :%s/shutdown/reboot/g
  ```
2）清空所有字符
```
  :%d
  ```
3）不保存退出
```
  q!
  ```
19. 1到10数字相加，写出shell脚本
```
  #!/bin/bash
  j=0
  for((i=1;i<=10;i++));do
      j=$[j+i ]
  done
  echo $j
  ```
20. 常见shell有哪些？缺省的是哪个？
```
  /bin/sh   /bin/bash   /bin/ash    /bin/bsh    /bin/csh   /bin/tcsh    /sbin/nologin
  ```
21. Shell循环语句有哪些？
```
  for    while    until
  ```
22. 用SHELL模拟LVS，脚本怎么写
```
  /sbin/iptable -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to192.168.1.11-192.168.1.12
```
23. 找出系统内大于50k，小于100k的文件，并删除它们。
```
  #!/bin/bash
  file=`find / -size +50k -size -100k`
  for i in $file;do
      rm -rf $i
  done
```
24. 脚本（如：目录dir1、dir2、dir3下分别有file1、file2、file2，请使用脚本将文件改为dir1_file1、dir2_file2、dir3_file3）
```
  #!/bin/bash
  file=`ls dir[123]/file[123]`
  for i in $file;do
      mv $i ${i%/*}/${i%%/*}_${i##*/}
  done
```
25. 将A 、B、C目录下的文件A1、A2、A3文件，改名为AA1、AA2、AA3.使用shell脚本实现。
```
  #!/bin/bash
  file=`ls [ABC]/A[123]`
  for i in $file;do
      mv $i ${i%/*}/A${i#*/}
  done
```
