# OpenWRT notes

Image features:

- target: x86_64
- rootfs (squashfs) size: 900MB
- standard kernel modules
- support for bash, less, vim-fuller, nginx
- rename to `Nextsecurity`

## Build environment

Prepare a Debian 11 machine:
```
useradd builder -s /bin/bash -m
apt install build-essential ccache ecj fastjar file g++ gawk gettext git java-propose-classpath libelf-dev libncurses5-dev libncursesw5-dev libssl-dev python python2.7-dev python3 unzip wget  python3-setuptools python3-dev rsync subversion swig time xsltproc zlib1g-dev linux-headers-amd64 make sudo libpam-dev  liblzma-dev libsnmp-dev
```

Switch to `builder` user then clone the repo and apply special config:
```
su - builder

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

## Publish feeds

Publish feeds on Debian 11 with quick and dirty hack (do not do it on production):
```
apt-get install lighttpd
system enable --now lighttpd
chgrp builder /var/www/html
chmod g+w /var/www/html
```

After the build, from `builder` user:
```
cp -r bin/targets/x86/64/packages/* /var/www/html/core/
cp -r bin/packages/x86_64/* /var/www/html/
```

## JSON-RPC CORS

Install required packages:
```
opkg update
opkg install luci-mod-rpc luci-lib-ipkg
```

Patch to enable CORS:
```diff
# diff -u /etc/nginx/conf.d/luci.locations.ori /etc/nginx/conf.d/luci.locations
--- /etc/nginx/conf.d/luci.locations.ori    2022-04-20 15:54:21.000000000 +0000
+++ /etc/nginx/conf.d/luci.locations    2022-04-20 15:52:31.000000000 +0000
@@ -1,4 +1,6 @@
 location /cgi-bin/luci {
+        add_header 'Access-Control-Allow-Origin' '*' always;
+        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
+        add_header 'Access-Control-Allow-Headers' 'content-type' always;
+
         index  index.html;
         include uwsgi_params;
         uwsgi_param SERVER_ADDR $server_addr;
@@ -6,6 +8,8 @@
         uwsgi_pass unix:////var/run/luci-webui.socket;
 }
 location ~ /cgi-bin/cgi-(backup|download|upload|exec) {
+        add_header 'Access-Control-Allow-Origin' '*' always;
+
         include uwsgi_params;
         uwsgi_param SERVER_ADDR $server_addr;
         uwsgi_modifier1 9;
@@ -17,6 +21,8 @@
 }
 
 location /ubus {
+        add_header 'Access-Control-Allow-Origin' '*' always;
+
         ubus_interpreter;
         ubus_socket_path /var/run/ubus/ubus.sock;
         ubus_parallel_req 2;
```

Example of simple plugin for `rpcd`:
```
local methods = {
	listObjects = {
		-- args = { type = "type" },
		call = function(args)
			result = {}
			result["hosts"] = {}
			paths, num = fs.glob("/etc/config/objects/hosts/*.ipset")
			for path in paths do
				name = fs.basename(path)
				table.insert(result["hosts"], string.sub(name, 0, -7))
			end
			return result
		end
	},
}
```

## netify-fwa

How to build netify-fwa on Debian 11:
```
apt install -y autoconf automake make libtool pkg-config
git clone https://gitlab.com/netify.ai/public/netify-fwa.git
cd netify-fwa/
./autogen.sh
./configure --prefix=/usr --includedir=${prefix}/include --mandir=${prefix}/share/man --infodir=${prefix}/share/info --sysconfdir=/etc --localstatedir=/var
./deploy/openwrt/package/make-package.sh 0.g$(git rev-parse --short HEAD)
```


# Links

- luci translations: https://forum.openwrt.org/t/solved-translation-of-packages-for-luci/30534/17
- fw4 status:  https://forum.openwrt.org/t/firewall4-is-now-default-is-this-totally-transparent-to-users/117753/32
