# Build

Prepare a Debian 11 machine:
```
useradd builder -s /bin/bash -m
apt install build-essential ccache ecj fastjar file g++ gawk gettext git java-propose-classpath libelf-dev libncurses5-dev libncursesw5-dev libssl-dev python python2.7-dev python3 unzip wget  python3-setuptools python3-dev rsync subversion swig time xsltproc zlib1g-dev linux-headers-amd64 make sudo libpam-dev  liblzma-dev libsnmp-dev
```

Switch to `builder` user then clone the repo and apply special config:
```
su - builder

git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
oppure
wget https://github.com/openwrt/openwrt/archive/refs/tags/v22.03.0-rc1.tar.gz -O openwrt.tgz
mkidr openwrt
tar xvf openwrt.tgz --strip-components=1 -C openwrt 
cd openwrt
sed -i '/telephony/d' feeds.conf.default
sed -i 's/src-git-full/src-git/' feeds.conf.default
```

Prepare the build env:
```
./scripts/feeds update
./scripts/feeds install -a
```

Then copy `config` file to `openwrt` directory and rename it `.config`.
Finally, start the build:
```
make -j $(($(nproc)+1)) download
make -j1 V=sc toolchain/compile
make -j $(nproc) kernel_menuconfig
make -j1 V=sc package/feeds/packages/perl/host/{clean,compile}
make -j $(($(nproc)+1)) V=sc
make -j1 V=sc world
```
