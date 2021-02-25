------



# Docker使用教程

## 1 拉取镜像

   docker pull tomcat

## 2 启动tomcat容器

   docker run --name tomcat -p 8080:8080 -d 镜像名

## 3 复制启动tomcat容器中的三个常用目录 conf、webapps、logs 用于挂载宿主机

   docker cp tomcat:/usr/local/tomcat/webapps.dist tomcat

   docker cp tomcat:/usr/local/tomcat/conf tomcat

   docker cp tomcat:/usr/local/tomcat/logs tomcat

## 4 停止tomcat容器

   docker stop tomcat

## 5 移除tomcat

   docker rm tomcat

## 6 挂载宿主目录webapps、logs、conf 启动tomcat容器

   docker run --name tomcat -p 8080:8080 -v \$PWD/webapps:/usr/local/tomcat/webapps -v \$PWD/logs:/usr/local/tomcat/logs -v \$PWD/conf:/usr/local/tomcat/conf -d tomcat

## 7 部署项目

   将war包放在webapps下（如果不是ROOT.war，需要设置war包权限，否则报错）

## 8 将容器打包为镜像文件

   docker commit -a "runoob.com" -m "my apache" 容器名称或id 打包的镜像名称:标签

## 9 导出镜像

   docker save -o python_3.tar python:3

## 10 导入镜像

   docker load -i python_3.tar

## 11 测试
   
   http://45.76.203.215:8080/index.html

## 12 参考博客链接

   https://blog.csdn.net/CSDN877425287/article/details/106712312?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522159282606719724843328198%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=159282606719724843328198&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_ctr_v3-16-106712312.ecpm_v1_rank_ctr_v3&utm_term=docker+tomcat+%E6%8C%82%E8%BD%BD%E7%9B%AE%E5%BD%95

## 13 docker 容器连接 tomcat+mysql

   https://www.jianshu.com/p/b494c8ff1e8d