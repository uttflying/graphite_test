## 安装依赖程序：

    sudo  rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch
    vi /etc/yum.repos.d/elasticsearch.repo
      [elasticsearch-1.0]
      name=Elasticsearch repository for 1.0.x packages
      baseurl=http://packages.elasticsearch.org/elasticsearch/1.0/centos
      gpgcheck=1
      gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
      enabled=1

    sudo yum install nodejs npm java-1.7.0-openjdk elasticsearch -y
    sudo npm install -g karma

## 安装grafana及配置
    cd /home/wenba/software/tar/
    wget http://grafanarel.s3.amazonaws.com/grafana-1.8.0.tar.gz
    tar zxvf grafana-1.8.0.tar.gz -C /home/wenba/
    mv /home/wenba/grafana-1.8.0/ /home/wenba/grafana

#### 配置graphite和elasticsearch

    cd /home/wenba/grafana/
    cp config.sample.js config.js
    vi config.js 

#### 关闭ES远程执行脚本功能

    vi /etc/elasticsearch/elasticsearch.yml 添加如下一行
      script.disable_dynamic: true # 手动关闭ES远程执行脚本的功能，防止被挂马


#### 为Graphite添加CORS支持

一款设置CORS（Cross-Origin Resource Sharing）标头的应用，基于XmlHttpRequest，对管理Django应用中的跨域请求非常有帮助

    sudo pip install django-cors-headers configobj
    vi /home/wenba/graphite/webapp/graphite/app_settings.py
      INSTALLED_APPS里面添加corsheaders, MIDDLEWARE_CLASSES里面添加’corsheaders.middleware.CorsMiddleware’

## 配置httpd

#### 配置graphite-web
    sudo vi /etc/httpd/conf.d/graphite-vhost.conf
    
#### 配置grafana
    sudo vi /etc/httpd/conf.d/grafana.conf
    sudo htpasswd -c /etc/httpd/conf.d/grafana.htpasswd grafana     # grafana12345

## 启动服务

    sudo /sbin/chkconfig --add httpd
    sudo /sbin/chkconfig httpd on
    sudo /etc/init.d/httpd reload
    sudo /sbin/chkconfig --add elasticsearch
    sudo service elasticsearch start

## 访问服务

访问地址：http://10.10.0.1:8080/
