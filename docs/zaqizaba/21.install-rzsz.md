## 源码安装
```
wget http://www.ohse.de/uwe/releases/lrzsz-0.12.20.tar.gz
```

```
tar zxvf lrzsz-0.12.20.tar.gz
```

```
cd lrzsz-0.12.20
./configure && make && make install
```

安装过程默认把lsz和lrz安装到了/usr/local/bin/目录下,再做一次软链接
```
cd /usr/bin
ln -s /usr/local/bin/lrz rz
ln -s /usr/local/bin/lsz sz
```

## yum安装
```
yum install -y lrzsz
```