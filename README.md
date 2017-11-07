# hello-express-boshrelease

How to create this boshrelease is following

## init

```
bosh init-release --dir=hello-express-boshrelease --git

cd hello-express-boshrelease
echo '*.tar.xz' >> .gitignore
echo '*.zip' >> .gitignore
```

## node package

```
bosh generate-package node
```

### add blob

```
curl -L -O -J https://nodejs.org/dist/v8.9.0/node-v8.9.0-linux-x64.tar.xz
bosh add-blob node-v8.9.0-linux-x64.tar.xz node/node-v8.9.0-linux-x64.tar.xz
rm -f node-v8.9.0-linux-x64.tar.xz
```

### spec

```
cat <<"EOF" > packages/node/spec
---
name: node

dependencies: []

files:
- node/node-v8.9.0-linux-x64.tar.xz
EOF
```

### packaging


```
cat <<"EOF" > packages/node/packaging
set -e

tar xf node/node-*.tar.xz
mv node-*/* ${BOSH_INSTALL_TARGET}/
EOF
```

## hello-express package

```
bosh generate-package hello-express
```

### add-blob


```
git submodule add https://github.com/making/hello-express.git src/hello-express

cd src/hello-express
npm install
zip -r ../../hello-express-offline.zip *
cd ../..
bosh add-blob hello-express-offline.zip hello-express/hello-express-offline.zip
rm -f hello-express-offline.zip
```

### spec

```
cat <<"EOF" > packages/hello-express/spec
---
name: hello-express

dependencies: []

files:
- hello-express/hello-express-offline.zip
EOF
```

### packaging


```
cat <<"EOF" > packages/hello-express/packaging
set -e

unzip hello-express/hello-express-offline.zip -d ${BOSH_INSTALL_TARGET}/
EOF
```

## hello-express job

```
bosh generate-job hello-express
```

### ctl

```
cat <<"EOF" > jobs/hello-express/templates/ctl
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/hello-express
LOG_DIR=/var/vcap/sys/log/hello-express

PIDFILE=$RUN_DIR/hello-express.pid
RUNAS=vcap

export PATH=/var/vcap/packages/node/bin:$PATH

function pid_exists() {
  ps -p $1 &> /dev/null
}

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R $RUNAS:$RUNAS $RUN_DIR $LOG_DIR

    echo $$ > $PIDFILE

    exec chpst -u $RUNAS:$RUNAS \
      node /var/vcap/packages/hello-express/app.js \
      >>$LOG_DIR/server.stdout.log 2>>$LOG_DIR/server.stderr.log
    ;;

  stop)
    PID=$(head -1 $PIDFILE)
    if [ ! -z $PID ] && pid_exists $PID; then
      kill $PID
    fi
    while [ -e /proc/$PID ]; do sleep 0.1; done
    rm -f $PIDFILE
    ;;

  *)
  echo "Usage: ctl {start|stop|console}" ;;
esac
exit 0
EOF

chmod +x jobs/hello-express/templates/ctl
```

### monit


```
cat <<"EOF" > jobs/hello-express/monit
check process hello-express
  with pidfile /var/vcap/sys/run/hello-express/hello-express.pid
  start program "/var/vcap/jobs/hello-express/bin/ctl start"
  stop program "/var/vcap/jobs/hello-express/bin/ctl stop"
  group vcap
EOF
```

### spec


```
cat <<"EOF" > jobs/hello-express/spec
---
name: hello-express
templates:
  ctl: bin/ctl

packages:
- node
- hello-express

properties: {}
EOF
```

## config/final.yml

```
cat <<"EOF" > config/final.yml
---
blobstore:
  provider: local
  options:
    blobstore_path: /tmp/hello-express
final_name: hello-express
EOF
```

## create release

```
bosh create-release --name=hello-express --force --timestamp-version --tarball=/tmp/hello-express-boshrelease.tgz
bosh upload-release /tmp/hello-express-boshrelease.tgz
```

## manifest


```
cat <<"EOF" > manifest.yml
---
name: hello-express

releases:
- name: hello-express
  version: latest

stemcells:
- os: ubuntu-trusty
  alias: trusty
  version: latest

instance_groups:
- name: hello-express
  jobs:
  - name: hello-express
    release: hello-express
    properties: {}
  instances: 1
  stemcell: trusty
  azs: [z1]
  vm_type: default
  networks:
  - name: default

update:
  canaries: 1
  max_in_flight: 3
  canary_watch_time: 30000-600000
  update_watch_time: 5000-600000
EOF
```

## deploy

```
bosh deploy -d hello-express manifest.yml
```
