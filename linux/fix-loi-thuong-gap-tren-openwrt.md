---
layout: page.njk
title: Fix lỗi thường gặp trên OpenWRT
description: Một số tips, trick cho các thiết bị chạy trên OpenWRT
cover: https://linuxiac.b-cdn.net/wp-content/uploads/2021/09/openwrt.png
author: Van Thanh
time: 06/13/2022
tags:
  - linux
  - openwrt
---
Nếu các bạn chưa biết thì OpenWRT là hệ điều hành mã nguồn mở dựa trên Linux được thiết kế cho các thiết bị như router, switch, wifi access point, ... Nhờ việc là mã nguồn mở mà có rất nhiều add-on được viết cho OpenWRT để chạy thêm nhiều tính năng của Linux như Docker, KVM-Qemu, VLAN, proxy, ...

Do được viết dựa trên core musl của Linux nên sẽ có nhiều thành phần bị cắt giảm tối đa với mục đích đủ duong lượng trên các thiết bị cấu hình thấp, sẽ có một vài lỗi vặt xảy ra hoặc do không tương thích. Trong bài viết này mình sẽ tổng hợp một vài lỗi mà mình gặp phải trên thiết bị của mình nhé.

## Thiết bị

Hiện tại thì mình đang sử dụng máy tính mini dùng chip Intel Celeron J4125, máy có 4 cổng mạng 2.5G dựa trên chịp Realtek RTL8125B. Bạn có thể xem review sơ qua của mình ở [đây](https://voz.vn/t/mini-review-router-j4125-rtl8125-4-cong-2-5g.443542/).

Bản OpenWRT của mình là do các [pháp sư trung hoa](https://github.com/HoldOnBro/Actions-OpenWrt) build dựa trên OpenWRT 21 mới nhất.

<img src='https://i.imgur.com/Ni2axOp.png' style="max-width: 700px" /><br /><br />

## Docker

Dù mình đã sửa file `/etc/docker/daemon.json` để chuyển data của Docker sang phân vùng mới, thì khi tải image về, docker vẫn tải về phân vùng hệ thống (200MB) nên báo lỗi hết ổ cứng. Để Fix thì mọi người dùng symlink link cái folder tạm của docker sang phân vùng mới là được.
`ln -s /mount/cua/phan/vung/moi/docker_temp /var/lib/docker`

## Samba

Nếu gặp vấn đề với samba4 ko kết nối được, thì vào system -> startup để disable vào stop server ksmbd, sau đó restart service samba4.

## Crontab

Crontab của openwrt khá ngu, dùng @hourly @daily không chạy được đâu, mọi người dùng kiểu cũ nhé: `0 * * * *`

## DNS

Khi trỏ domain vào đây thì mọi người nhớ chỉnh cả `A` record vào ipv4, `AAAA` record vào ipv6, sau đó vào `Network` -> `DHCP & DDNS` để whitelist domain đó hoặc tắt hẳn `Rebind protection` đi, như thế các thiết bị trong LAN sẽ vào được domain như bình thường.

## Grub

Grub có mội lỗi nếu cắm USB hoặc SSD thứ 2 thì ko boot được, mọi người tạm tháo usb ra để vào, sau đó sửa file `/boot/grub/grub.cfg`. Sửa `set root='(hd0,gpt1)'` thành set `root='(hd1,gpt1)'` rồi cắm usb/ssd vào reboot bình thường.

## KVM/qEMU

OpenWRT không chỉ cho cài container trên Docker, mà còn có thể chạy máy ảo KVM/qEMU luôn, passthrough  usb, ssd, gpu bình thường.

Đầu tiên bạn cần cài đặt các gói tin cần thiết sau:
```
opkg install kmod-tun qemu-bridge-helper qemu-x86_64-softmmu kmod-kvm-intel
```

Tạo 1 ổ cứng cho máy ảo, ở đây mình tạo 8Gb làm  ví dụ:
```
qemu-img create -f qcow2 debian.qcow2 8G
```

Chạy thử máy ảo với file cài iso:
```
qemu-system-x86_64 -m 512 -nic user -boot d -cdrom ./iso/debian-11.1.0-amd64-netinst.iso -hda debian.qcow2
```

Để có thể dùng VNC lúc cài đặt, bạn cần mở port đên OpenWRT từ máy tính của bạn, ví dụ OpenWRT mình chạy ở 192.168.2.1 thì mình sẽ gõ lệnh này từ máy tính, sau đó dùng VNC đến `localhost:5900`
```
ssh -nfNT -L 5900:127.0.0.1:5901 root@192.168.2.1
```

Sau khi cài đặt xong, nếu bạn muốn giữ MAC address không bị đổi thì dùng lệnh này:
```
qemu-system-x86_64 -boot d -smp 2 -m 1G -enable-kvm \
	-cdrom ./iso/debian-11.1.0-amd64-netinst.iso \
	-hda debian.qcow2 \
	-device virtio-net-pci,mac=aa:88:04:fd:20:ba,netdev=br0 \
  -netdev bridge,br=br-lan,id=br0
```

Nếu bạn muốn máy ảo khởi động cùng máy như một service của OpenWRT, bạn cần tạo file service cho thiết bị. Ví dụ ở đây mình đặt tên service là qemu chẳng hạn.
```
nano /etc/init.d/qemu
```
Điền vào file như sau:
```
#!/bin/sh /etc/rc.common
 
START=99
STOP=1
 
qemu_pidfile="/var/run/qemu-debian.pid"
 
start() {
qemu-system-x86_64 -enable-kvm -cpu host -smp 2 -m 1G \
	-hda /nas/VM/debian.qcow2 \
	-device virtio-net-pci,mac=aa:88:04:fd:20:ba,netdev=br0 \
	-netdev bridge,br=br-lan,id=br0 \
	-qmp tcp:127.0.0.1:4444,server,nowait \
	-daemonize &> /var/log/qemu-debian.log
 
/usr/bin/pgrep qemu-system-x86_64 > $qemu_pidfile
echo "QEMU: Started VM with PID $(cat $qemu_pidfile)."
}
 
stop() {
echo "QEMU: Sending 'system_powerdown' to VM with PID $(cat $qemu_pidfile)."
nc localhost 4444 <<QMP 
{ "execute": "qmp_capabilities" } 
{ "execute": "system_powerdown" } 
QMP

if [ -e $qemu_pidfile ]; then
	if [ -e /proc/$(cat $qemu_pidfile) ]; then
		echo "QEMU: Waiting for VM shutdown."
		while [ -e /proc/$(cat $qemu_pidfile) ]; do sleep 1s; done
		echo "QEMU: VM Process $(cat $qemu_pidfile) finished."
	else
		echo "QEMU: Error: No VM with PID $(cat $qemu_pidfile) running."
	fi
 
	rm -f $qemu_pidfile
else
	echo "QEMU: Error: $qemu_pidfile doesn't exist."
fi
}
```
```
chmod +x /etc/init.d/qemu
```

Chú ý nếu bạn có nhiều VM cùng chạy thì nhớ đổi port `4444` thành các port khác nhau cho mỗi VM.

Giờ chỉ cần lưu lại là bạn có thể điều khiển trong `System` => `Startup` trên Website của OpenWRT.