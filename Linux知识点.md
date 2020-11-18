#Linux知识点记录
####1.使用winscp上传文件到ubuntu
 出现软件造成的连接错误提示时
 ```linux
sudo apt-get remove vsftpd
sudo rm /etc/pam.d/vsftpd
sudo apt-get install vsftpd 
```
这是因为ubuntu启用了PAM,所在用到vsftp时需要用到/etc/pam.d/vsftpd这个文件（默认源码安装的不会有这个文件），因此除了匿名用户外本地用户无法登录。所以只要删除了就可以了。
####2.某个服务的进程，使用PS
ps -ef|grep uwsgi
ps -ef|grep python
kill -9 [进程ID]