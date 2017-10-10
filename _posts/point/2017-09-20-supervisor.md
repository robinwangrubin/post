---
layout: post
title: supervisor进程守护
categories: Point
---

```
mkdir -p /etc/supervisor/conf
mkdir -p /var/log/supervisor
yum install python-setuptools-devel
easy_install supervisor
echo_supervisord_conf > /etc/supervisor/supervisord.conf.bak
cat >/etc/supervisor/supervisord.conf<<EOF
[unix_http_server]
file=/var/run/supervisor.sock   ; the path to the socket file
[supervisord]
logfile=/var/log/supervisor/supervisord.log ; main log file; default $CWD/supervisord.log
logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
logfile_backups=10           ; # of main logfile backups; 0 means none, default 10
loglevel=info                ; log level; default info; others: debug,warn,trace
pidfile=/var/run/supervisord.pid ; supervisord pidfile; default supervisord.pid
nodaemon=false               ; start in foreground if true; default false
minfds=65535                  ; min. avail startup file descriptors; default 1024
minprocs=65535                 ; min. avail process descriptors;default 200
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket
[include]
files = /etc/supervisor/conf/*.ini
EOF
supervisord -c /etc/supervisor/supervisord.conf
echo "supervisord -c /etc/supervisor/supervisord.conf" >>/etc/rc.local 
```

-----------------

end
