## 依赖关系解决：

    sudo yum -y install bitmap bitmap-fonts Django pycairo python-devel python-ldap python-memcached mod_wsgi mod_python python-sqlite2 glibc-devel gcc gcc-c++ git openssl-devel  httpd memcached python-hashlib django-tagging  python-simplejson pip

#### 安装Twisted和zope.interface

Twisted需要配合zope.interface 3.6以上版本使用

    wget --no-check-certificate https://pypi.python.org/packages/source/z/zope.interface/zope.interface-4.1.1.tar.gz
     tar xvf zope.interface-4.1.1.tar.gz -C ../src/
     cd ../src/zope.interface-4.1.1/
     sudo python setup.py install
    wget --no-check-certificate https://pypi.python.org/packages/source/T/Twisted/Twisted-14.0.1.tar.bz2
     tar xvf Twisted-14.0.1.tar.bz2 -C ../src/
     cd ../src/Twisted-14.0.1/
     sudo python setup.py install

## 安装步骤

mkdir /home/test/graphite
cd /home/test/graphite
sudo pip install https://github.com/graphite-project/ceres/tarball/master
sudo pip install carbon --install-option="--prefix=/home/test/graphite" --install-option="--install-lib=/home/test/graphite/lib"
sudo pip install whisper （后面千万不要添加参数，否则会导致whisper安装失败的）
sudo pip install graphite-web --install-option="--prefix=/home/test/graphite" --install-option="--install-lib=/home/test/graphite/webapp"

sudo pip install daemonize
vi /home/test/graphite/lib/carbon/util.py
     改from twisted.scripts._twistd_unix import daemonize 变成 import daemonize

## 初始化配置：

cd /home/test/graphite/conf
cp carbon.conf.example carbon.conf
cp storage-schemas.conf.example storage-schemas.conf
cp graphite.wsgi.example graphite.wsgi
cp /home/test/graphite/examples/example-graphite-vhost.conf /etc/httpd/conf.d/graphite-vhost.conf
vi  /etc/httpd/conf.d/graphite-vhost.conf
     WSGISocketPrefix /home/test/graphite/wsgi
     同时修改所有opt为home/test，并修改User和Group都为test
chown -R test.test /var/run/httpd/
mkdir /home/test/graphite/wsgi
vi /home/test/graphite/conf/graphite.wsgi
     修改为sys.path.append('/home/test/graphite/webapp')

#### 为了防止whisper的产生数据过多，占用/分区的磁盘空间，需要转移至/data目录下

cd /home/test/graphite/storage/
mkdir -pv /data/whisper_new/
ln -s /data/whisper_new/  whisper

## 初始化数据库并启动服务：

sudo chown -R test.test /home/test/graphite
cd /home/test/graphite/webapp/graphite/
cp local_settings.py.example local_settings.py
vi local_settings.py
     # 修改GRAPHITE_ROOT、WHISPER_DIR、DATABASES等参数
     GRAPHITE_ROOT = '/home/test/graphite'
     WHISPER_DIR = '/home/test/graphite/storage/whisper'
     DATABASES = {
        'default': {
            'NAME': '/home/test/graphite/storage/graphite.db',
            'ENGINE': 'django.db.backends.sqlite3',
            'USER': '',
            'PASSWORD': '',
            'HOST': '',
            'PORT': ''
        }
    }

python manage.py syncdb # 注：初始化数据库时设定用户test，邮箱jie.yu@test100.com，密码test12345，这也是web界面的登录账号和密码

sudo /etc/init.d/httpd restart
cd /home/test/graphite/
./bin/carbon-cache.py start


## 测试

#### 手动生成数据
echo "local.random.diceroll 4 `date +%s`" | nc -q0 127.0.0.1 2003

#### 测试程序生成数据：
cd /home/test/graphite/examples/
python example-client.py


## 启动多个carbon-cache，并使用carbon-relay调度metrics至各carbon-cache

vi /home/test/graphite/conf/carbon.conf
     添加instance-b和instance-c
     [cache:b]
     LINE_RECEIVER_PORT = 2103
     PICKLE_RECEIVER_PORT = 2104
     CACHE_QUERY_PORT = 7102

     [cache:c]
     LINE_RECEIVER_PORT = 2203
     PICKLE_RECEIVER_PORT = 2204
     CACHE_QUERY_PORT = 7202
#### 分别启动instance-b和instance-c

cd /home/test/graphite
bin/carbon-cache.py --instance=b start
bin/carbon-cache.py --instance=c start

#### 配置relay并启动

vi /home/test/graphite/conf/carbon.conf修改relay配置
[relay]
LINE_RECEIVER_INTERFACE = 0.0.0.0
LINE_RECEIVER_PORT = 2013
PICKLE_RECEIVER_INTERFACE = 0.0.0.0
PICKLE_RECEIVER_PORT = 2014
DESTINATIONS = 127.0.0.1:2004:a, 127.0.0.1:2104:b, 127.0.0.1:2204:c

##### 同时可定义relay规则：vi /home/test/graphite/conf/relay-rules.conf

[servers]
pattern = ^servers\.
destinations = 127.0.0.1:2004:a, 127.0.0.1:2104:b, 127.0.0.1:2204:c
[default]
default = true
destinations = 127.0.0.1:2004:a, 127.0.0.1:2104:b, 127.0.0.1:2204:c

##### 启动relay程序：

cd /home/test/graphite
./bin/carbon-relay.py start
然后修改各diamond.conf的配置文件中连接carbon的端口号为carbon-relay的端口2013和2014

## 其它

#### 检查graphite的依赖关系

wget http://launchpad.net/graphite/0.9/0.9.9/+download/graphite-web-0.9.9.tar.gz
tar zxvf graphite-web-0.9.9.tar.gz -C ../src
cd /home/test/software/src/graphite-web-0.9.9/
python check-dependencies.py
