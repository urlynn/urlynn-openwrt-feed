# OpenWrt 从编译到 ProxmoxVE lxc 容器部署

## OpenWrt 编译篇

### 编辑环境准备


1. 准备 `Alpine rootfs`
   
   ```bash
   bash -c "
     mkdir -p /alpine-root
     mkdir -p /root/openwrt
     cd /alpine-root
     curl -OL https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-minirootfs-3.21.3-x86_64.tar.gz
     tar xzf alpine-minirootfs-*.tar.gz
     rm alpine-minirootfs-*.tar.gz
   "
   ```
   
2. `Container` 入口脚本

   ```bash
   bash -c '
   cat > /usr/local/bin/openwrt << '\''EOF'\''
   #!/bin/sh
   export SYSTEMD_COLORS=0
   exec systemd-nspawn -q -D /alpine-root \
     --bind /root/openwrt:/openwrt \
     --bind /tmp:/tmp \
     --chdir=/openwrt \
     "\$@"
   EOF
   chmod +x /usr/local/bin/openwrt
   '
   ```
      
3. 软件包安装
   ```shell
   apk add argp-standalone asciidoc bash bc binutils bzip2 cdrkit coreutils \
     diffutils elfutils-dev findutils flex musl-fts-dev g++ gawk gcc gettext git \
     grep gzip intltool libxslt linux-headers make musl-libintl musl-obstack-dev \
     ncurses-dev openssl-dev patch perl python3-dev rsync tar \
     unzip util-linux wget zlib-dev
   ```

4. 仓库克隆

   ```shell
   # Download and update the sources
   git clone https://git.openwrt.org/openwrt/openwrt.git
   cd openwrt
   git pull
    
   # Select a specific code revision
   git branch -a
   git tag
   git checkout v24.10.7
   ```

5. Update the feeds

   > `feeds.conf.my`
   >
   > ```shell
   > src-git openclash https://github.com/vernesong/OpenClash.git
   > src-git my_feed https://github.com/znwuyi/urlynn-openwrt-feed.git
   > ```

   ```shell
   cat feeds.conf.default feeds.conf.my > feeds.conf
   ./scripts/feeds update -a
   ./scripts/feeds install -a -f
   ```

6. Choose package

   ```shell
   make menuconfig # Configure the firmware image
   ```

   ```shell
   -> Target System 
   	-> x86
   -> Subtarget
   	-> x86_64
   -> Target Profile
   	-> Generic x86/64
   -> Target Images
   	-> [*] tar.gz #仅勾选`tar.gz`
   -> Advanced configuration options (for developers)
   	-> Toolchain Options
   		-> GCC compiler Version
   			-> gcc 14.x  
   -> Base system
   	-> < > dnsmasq
   	-> -*- dnsmasq-full 
   -> LuCI
   	-> 1. Collections
   		-> <*> luci-nginx
   	-> 2. Modules
   		-> <*> luci-compat
   		-> Translations
   			-> <*> Chinese Simplified (zh_Hans) 
   	-> 3. Applications
   		-> <*> luci-app-openclash
   		-> <*> luci-app-upnp
   	-> 4. Themes
   		-> <*> luci-theme-argon
   		-> <*> luci-theme-alpha
   -> Utilities
   	-> Editors
   		-> <*> nano-full
   		-> <*> vim-full
   	-> Shells
   		-> <*> fish
   ```

7. 开始编译

   > llvm版Gentoo Linux需修改gcc路径，其他操作系统忽略此步
   >
   > ```shell
   > rm ~/openwrt/staging_dir/host/bin/gcc
   > rm ~/openwrt/staging_dir/host/bin/g++
   > ln -s /usr/bin/gcc ~/openwrt/staging_dir/host/bin/gcc
   > ln -s /usr/bin/g++ ~/openwrt/staging_dir/host/bin/g++
   > ```

   ```shell
   make download -j24 V=s
   make V=s -j(nproc); or make V=s -j1
   ```

   编译完成后，生成的文件路径为`~/openwrt/bin/targets/x86/64/openwrt-x86-64-generic-rootfs.tar.gz`

   ```shell
   #拷贝至ProxmoxVE
   scp /home/znwuyi/openwrt/bin/targets/x86/64/openwrt-x86-64-generic-rootfs.tar.gz root@192.168.1.10:/var/lib/vz/template/cache/
   ```

> **参考文献**
>
> 1. [Build System Usage](https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem)

## ProxmoxVE 容器篇

### 自动化脚本

> `new.sh`
>
> ```shell
> #!/usr/bin/fish
> 
> # ------------------
> # 配置区（用户可编辑）
> # ------------------
> set -gx CTID 101
> set -gx TEMPLATE "openwrt-x86-64-generic-rootfs.tar.gz"
> set -gx NET0_TYPE "phys"
> set -gx NET0_LINK "enp1s0"
> set -gx NET0_NAME "enp1s0"
> set -gx NET1_TYPE "veth"
> set -gx NET1_LINK "vmbr0"
> set -gx NET1_NAME "enp2s0v0"
> set -gx PPPOE_USER "0730*******0"
> set -gx PPPOE_PASS "1******9"
> 
> # ------------------
> # 主流程
> # ------------------
> pct create $CTID \
>   local:vztmpl/$TEMPLATE \
>   --rootfs local-lvm:1 \
>   --ostype unmanaged \
>   --hostname OpenWrt \
>   --cores 4 \
>   --memory 1024 \
>   --swap 0
> 
> envsubst < lxc.conf.template | sudo tee -a "/etc/pve/lxc/$CTID.conf"
> pct start $CTID || exit 1
> echo "等待容器初始化..."
> sleep 10  
> echo "容器初始化完成"
> lxc-attach -n $CTID -- cp /etc/config/network /etc/config/network.bak
> envsubst < network.conf.template | lxc-attach -n $CTID -- sh -c 'cat > /etc/config/network'
> lxc-attach -n $CTID -- /etc/init.d/network restart
> echo "容器 $CTID 部署完成！"
> ```
>
> 

> `lxc.conf.template`
>
> ```shell
> tags: route
> onboot: 0
> features: nesting=1
> lxc.net.0.type: ${NET0_TYPE}
> lxc.net.0.link: ${NET0_LINK}
> lxc.net.0.name: ${NET0_NAME}
> lxc.net.0.flags: up
> lxc.net.1.type: ${NET1_TYPE}
> lxc.net.1.link: ${NET1_LINK}
> lxc.net.1.name: ${NET1_NAME}
> lxc.net.1.flags: up
> lxc.include: /usr/share/lxc/config/openwrt.common.conf
> lxc.cgroup2.devices.allow: c 108:0 rwm
> lxc.cgroup2.devices.allow: c 10:200 rwm
> lxc.mount.entry: /dev/ppp dev/ppp none bind,create=file
> lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
> #lxc.mount.entry: /dev/net dev/net none bind,create=dir
> #lxc.mount.auto: proc:mixed sys:ro cgroup:mixed
> #hookscript: local:snippets/hookscript.pl
> lxc.cap.drop:
> ```

> `network.conf.template`
>
> ```shell
> config interface 'loopback'
>   option device 'lo'
>   option proto 'static'
>   option ipaddr '127.0.0.1'
>   option netmask '255.0.0.0'
> 
> config globals 'globals'
>   option packet_steering '1'
> 
> config interface 'lan'
>   option device '${NET1_NAME}'
>   option proto 'static'
>   option ipaddr '192.168.1.1'
>   option netmask '255.255.255.0'
>   option ip6assign '64'
>   option delegate '0'
>   option ip6ifaceid 'eui64'
> 
> config interface 'wan'
>   option device '${NET0_NAME}'
>   option proto 'pppoe'
>   option username '${PPPOE_USER}'
>   option password '${PPPOE_PASS}'
> ```

