---
layout: post
title: Flask deployment antics with CentOS7
---

NOTE!
This was a waste of time... if this topic interests you, look to my next post
/NOTE!


No better time to start a github blog than while waiting for python to compile from source on a new linode instance.

My first shot at deploying my pet project flask-python app to my production vps was discouraging.
Rathar than chase rabbits on that box, I'm deploying a test linode vps.  Thanks linode for the new $5/mo boxes!
While unlikely, I can hose this box while hacking out a deployment strategy.

Centos7 doesn't ship with the latest python version, of course.  The new VPS is getting a python environment compiled from source within an unprivileged user home directory.
That py install will be used to create a virtualenv that apache will run WSGI under.
Yes, i'm old so I still love apache.  

Simple to get started:

```
yum -y install gcc gcc-devel gcc-c++ gcc-c++-devel

yum install openssl-devel bzip2-devel expat-devel gdbm-devel readline-devel sqlite-devel

mkdir -p ~/usr/local
wget https://www.python.org/ftp/python/3.6.0/Python-3.6.0.tgz
tar -xzf Python-3.6.0.tgz

cd Python-3.6.0
./configure --enable-optimizations

make altinstall prefix=$HOME/usr/local exec-prefix=$HOME/usr/local
```

And compiling is done.  Get to work.
