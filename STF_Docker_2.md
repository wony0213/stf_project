```
docker run -d --name rethinkdb -v /srv/rethinkdb:/data --net host rethinkdb rethinkdb --bind all --cache-size 8192 --http-port 8090

docker run -d --name stf-migrate --net host  openstf/stf stf migrate

docker run -d --name adbd --privileged -v /dev/bus/usb:/dev/bus/usb --net host sorccu/adb:latest




docker run -d --name stf-triproxy-app001 --net host  openstf/stf stf triproxy app001 --bind-pub tcp://127.0.0.1:7111 --bind-dealer tcp://127.0.0.1:7112 --bind-pull tcp://127.0.0.1:7113

docker run -d --name stf-triproxy-dev001 --net host  openstf/stf stf triproxy dev001 --bind-pub tcp://127.0.0.1:7114 --bind-dealer tcp://127.0.0.1:7115 --bind-pull tcp://127.0.0.1:7116

docker run -d --name stf-processor-proc001 --net host  openstf/stf stf processor proc001 --connect-app-dealer tcp://127.0.0.1:7112 --connect-dev-dealer tcp://127.0.0.1:7115

docker run -d --name stf-processor-proc002 --net host  openstf/stf stf processor proc002 --connect-app-dealer tcp://127.0.0.1:7112 --connect-dev-dealer tcp://127.0.0.1:7115

docker run -d --name stf-reaper-reaper001 --net host  openstf/stf stf reaper reaper001 --connect-push tcp://127.0.0.1:7116 --connect-sub tcp://127.0.0.1:7111

docker run -d --name stf-provider-provider001 --net host  openstf/stf stf provider --name wony-0213deMacBook-Pro.local --min-port 7400 --max-port 7700 --connect-sub tcp://127.0.0.1:7114 --connect-push tcp://127.0.0.1:7116 --group-timeout 900 --public-ip localhost --storage-url http://localhost:7100/ --adb-host 127.0.0.1 --adb-port 5037 --vnc-initial-size 600x800 local

docker run -d --name stf-auth-mock --net host  openstf/stf stf auth-mock --port 7120 --secret kute kittykat --app-url http://localhost:7100/

docker run -d --name stf-app --net host  openstf/stf stf app --port 7105 --secret kute kittykat --auth-url http://localhost:7100/auth/mock/ --websocket-url http://localhost:7110/

docker run -d --name stf-api --net host  openstf/stf stf api --port 7106 --secret kute kittykat --connect-push tcp://127.0.0.1:7113 --connect-sub tcp://127.0.0.1:7111

docker run -d --name stf-websocket --net host  openstf/stf stf websocket --port 7110 --secret kute kittykat --storage-url http://localhost:7100/ --connect-sub tcp://127.0.0.1:7111 --connect-push tcp://127.0.0.1:7113

docker run -d --name stf-storage-temp --net host  openstf/stf stf storage-temp --port 7102

docker run -d --name stf-storage-plugin-image --net host  openstf/stf stf storage-plugin-image --port 7103 --storage-url http://localhost:7100/

docker run -d --name stf-storage-plugin-apk --net host  openstf/stf stf storage-plugin-apk --port 7104 --storage-url http://localhost:7100/

docker run -d --name stf-poorxy --net host  openstf/stf stf poorxy --port 7100 --app-url http://localhost:7105/ --auth-url http://localhost:7120/ --api-url http://localhost:7106/ --websocket-url http://localhost:7110/ --storage-url http://localhost:7102/ --storage-plugin-image-url http://localhost:7103/ --storage-plugin-apk-url http://localhost:7104/

```

