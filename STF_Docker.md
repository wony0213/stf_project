# 1、最简单的Docker部署

- 先把需要的镜像pull下来

```
docker pull openstf/stf:latest
docker pull sorccu/adb:latest
docker pull rethinkdb:latest
docker pull openstf/ambassador:latest
docker pull nginx:latest
```

- 分别在3个Docker上启动adb服务、rethinkdb服务和stf local服务（该Docker运行多个进程）

```
#先启动一个数据库
docker run -d --name test_rethinkdb -v /srv/rethinkdb:/data --net host rethinkdb rethinkdb --bind all --cache-size 8192 --http-port 8090
#再启动adb service
docker run -d --name test_adbd --privileged -v /dev/bus/usb:/dev/bus/usb --net host sorccu/adb:latest
#再启动stf
docker run -d --name test_stf --net host openstf/stf stf local --public-ip localhost
```
# 2、把不同的进程运行在不同的Docker上，然后通过nginx配置web服务。

## 启动Nginx

- nginx.conf配置文件如下，与官方给的配置文件对比，去掉了ssl的部分。参考：https://github.com/openstf/stf/blob/master/doc/DEPLOYMENT.md#stf-storage-plugin-apkservice

```
daemon off;
worker_processes 4;

events {
  worker_connections 1024;
}

http {
  upstream stf_app {
    server 10.0.5.47:3100 max_fails=0;
  }

  upstream stf_auth {
    server 10.0.5.47:3200 max_fails=0;
  }

  upstream stf_storage_apk {
    server 10.0.5.47:3300 max_fails=0;
  }

  upstream stf_storage_image {
    server 10.0.5.47:3400 max_fails=0;
  }

  upstream stf_storage {
    server 10.0.5.47:3500 max_fails=0;
  }

  upstream stf_websocket {
    server 10.0.5.47:3600 max_fails=0;
  }

  types {
    application/javascript  js;
    image/gif               gif;
    image/jpeg              jpg;
    text/css                css;
    text/html               html;
  }

  map $http_upgrade $connection_upgrade {
    default  upgrade;
    ''       close;
  }

  server {
    listen 80;
    server_name stf.test;
    keepalive_timeout 70;
    root /dev/null;

    resolver 127.0.0.1 valid=300s;
    resolver_timeout 10s;

    # Handle stf-provider@floor4.service
    location ~ "^/d/floor4/([^/]+)/(?<port>[0-9]{5})/$" {
      proxy_pass http://10.0.5.47:$port/;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_set_header X-Forwarded-For $remote_addr;
      proxy_set_header X-Real-IP $remote_addr;
    }

    location /auth/ {
      proxy_pass http://stf_auth/auth/;
    }

    location /s/image/ {
      proxy_pass http://stf_storage_image;
    }

    location /s/apk/ {
      proxy_pass http://stf_storage_apk;
    }

    location /s/ {
      client_max_body_size 1024m;
      client_body_buffer_size 128k;
      proxy_pass http://stf_storage;
    }

    location /socket.io/ {
      proxy_pass http://stf_websocket;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $http_x_real_ip;
    }

    location / {
      proxy_pass http://stf_app;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $http_x_real_ip;
    }
  }
}
```
- 启动Nginx命令

```
docker run -d -v /home/wony_0213/Develop/stf/nginx/nginx.conf:/etc/nginx/nginx.conf:ro --name nginx --net host nginx nginx
```

## 启动RethinkDB，启动完成后可以通过浏览器localhost：8080访问数据库

```
docker run -d --name rethinkdb -v /srv/rethinkdb:/data --net host rethinkdb rethinkdb --bind all --cache-size 8192 --http-port 8090
```

## 给数据库建表

```
docker run -d --name stf-migrate --net host  openstf/stf stf migrate
```

## 启动一堆storage，这些模块在stf截图和安装app时会用到

```
docker run -d --name storage-plugin-apk-3300 -p 3300:3000 --dns 8.8.8.8 openstf/stf stf storage-plugin-apk --port 3000   --storage-url http://stf.test/
docker run -d --name storage-plugin-image-3400  -p 3400:3000 --dns 8.8.8.8 openstf/stf stf storage-plugin-image --port 3000 --storage-url http://stf.test/
docker run -d  --name storage-temp-3500 -v /mnt/storage:/data -p 3500:3000 --dns 8.8.8.8 openstf/stf  stf storage-temp --port 3000  --save-dir /data
```

- 官方的配置文件中用的是Google的dns，可以改成自己的。
- 如果需要持久保存的数据，最好用-v对docker中的目录进行映射。
- 这里使用-p参数把docker内的端口绑定到主机端口，否则都是3000会冲突。

## 启动Triproxy

```
docker run -d  --name triproxy-app  --net host  openstf/stf  stf triproxy app   --bind-pub "tcp://*:7150"   --bind-dealer "tcp://*:7160"  --bind-pull "tcp://*:7170"
docker run -d --name triproxy-dev  --net host  openstf/stf  stf triproxy dev  --bind-pub "tcp://*:7250"  --bind-dealer "tcp://*:7260" --bind-pull "tcp://*:7270"
```

## 登录授权

```
docker run -d --name stf-auth3200 -e "SECRET=YOUR_SESSION_SECRET_HERE" -p 3200:3000 --dns 8.8.8.8 openstf/stf stf auth-mock --port 3000  --app-url http://stf.test/
```

- 默认情况下的mock登录随便输入name和email登录，当然可以自己接入自己的授权系统。

## 启动stf-app，web主界面，启动后就应该可以看stf.test了

```
docker run -d --name stf-app3100 --net host -e "SECRET=YOUR_SESSION_SECRET_HERE" -p 3100:3000 openstf/stf stf app --port 3100 --auth-url http://stf.test/auth/mock/ --websocket-url http://stf.test/
```

## stf-processor、websocket、reaper

```
#stf-processor
docker run -d --name stf-processor --net host openstf/stf  stf processor stf-processor.service --connect-app-dealer tcp://10.0.5.47:7160 --connect-dev-dealer tcp://10.0.5.47:7260

#websocket 3600
docker run -d --name websocket -e "SECRET=YOUR_SESSION_SECRET_HERE" --net host openstf/stf stf websocket --port 3600  --storage-url http://stf.test/ --connect-sub tcp://10.0.5.47:7150 --connect-push tcp://10.0.5.47:7170

#reaper
docker run -d --name reaper --net host openstf/stf stf reaper dev --connect-push tcp://10.0.5.47:7270  --connect-sub tcp://10.0.5.47:7150 --heartbeat-timeout 30000
```

## 启动provider

- provider的作为就是给主框架提供手机，注意，provider可以运行在同一台机器，也可以运行在其他机器，但是，这台机器一定要可直接访问的，因为手机屏幕其实就是从这里出来的。

- 在provider机器上一定要先运行adb的docker

```
#adb services
docker run -d --name adbd --privileged -v /dev/bus/usb:/dev/bus/usb --net host sorccu/adb:latest
```

- 运行provider

```
sudo docker run -d --name provider1 --net host openstf/stf stf provider --name "provider-test" --connect-sub tcp://10.0.5.47:7250  --connect-push tcp://10.0.5.47:7270  --storage-url http://10.0.5.47  --public-ip 10.0.5.47  --min-port=15000  --max-port=25000  --heartbeat-interval 20000 --screen-ws-url-pattern "ws://stf.test/d/floor4/<%= serial %>/<%= publicPort %>/"
```

## 配置

- 修改linux的hosts文件绑定stf.test到127.0.0.1，修改后可能需要手动生效
- 目前adbd做的是systemd启动，不知道是否有问题
- 目前手机连接有问题，正在解决。
- 使用第一种方式部署，手机可以正常使用
- 使用第二种方式部署，手机连接有显示不能使用，会断开连接。

## 只要加上域名和授权控制，就可以正常使用了

# 3、参考

- https://testerhome.com/topics/5206