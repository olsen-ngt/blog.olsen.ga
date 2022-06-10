---
layout: page.njk
title: Cài đặt Adguard Home trên Alpine Linux
description: Cách cài đặt Adguard Home trên Alpine Linux đơn giản nhất
cover: https://res.cloudinary.com/canonical/image/fetch/https://dashboard.snapcraft.io/site_media/appmedia/2020/04/1_1_L6wlYts.png
author: Van Thanh
time: 13/06/2022
tags:
  - linux
---
Hướng dẫn cài đặt chính thức của Adguard Home trên Github được viết cho hệ điều hành nhân Debian nên không thể sử dụng để cài đặt trên Alpine do không có Systemd.
Tuy nhiên chúng ta có thể dễ dàng chuyển qua sử dụng OpenRC để thay thế Như sau:

1. Sử dựng wget để tải file nén [Adguard Home](https://github.com/AdguardTeam/AdGuardHome/releases) từ Github về và giải nén ra `/opt/AdGuardHome`

2. Tạo file `/etc/init.d/AdguardHome` như sau:

```
#!/sbin/openrc-run
#
# openrc service-script for AdGuardHome
#
# place in /etc/init.d/
# start on boot: "rc-update add adguardhome"
# control service: "service adguardhome <start|stop|restart|status|checkconfig>"
#

description="AdGuard Home: Network-level blocker"

pidfile="/run/$RC_SVCNAME.pid"
command="/opt/AdGuardHome/AdGuardHome"
command_args="-s run"
command_background=true

extra_commands="checkconfig"

depend() {
  need net
  provide dns
  after firewall
}

checkconfig() {
  "$command" --check-config || return 1
}

stop() {
  if [ "${RC_CMD}" = "restart" ] ; then
    checkconfig || return 1
  fi

  ebegin "Stopping $RC_SVCNAME"
  start-stop-daemon --stop --exec "$command" \
    --pidfile "$pidfile" --quiet
  eend $?
}
```

3. Cho phép file được thực thi:

```
chmod +x /etc/init.d/AdguardHome
```

4. Bật service và cho phép khởi động lúc boot

```
rc-update add adguardhome
rc-service adguardhome start
```