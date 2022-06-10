---
layout: page.njk
title: Quản lý nhiệt độ thiết bị trên Proxmox 7
description: Thêm nhiệt độ CPU, GPU, Disk lên giao diện quản lý của Proxmox 7
cover: https://i.imgur.com/GyjF1ZA.png
author: Van Thanh
time: 06/13/2022
tags:
  - linux
  - proxmox
---

## Cài đặt các thư viện cần thiết
```
apt-update && apt-get install lm-sensors hddtemp -y
```

## Chỉnh sửa file giao diện của Proxmox

Trước khi chỉnh sửa, các bạn nhớ backup lại các file này đề phòng có lỗi gì xảy ra nhé.

Mở file `/usr/share/perl5/PVE/API2/Nodes.pm` và tìm `$dinfo` ở loanh quanh dòng 364:

```
$dinfo = df('/', 1);
```

Thêm vào bên dưới:

```
$res->{CPUtemperature} = `sensors`;
$res->{GPUtemperature} = `sensors`;
$res->{Nvmetemperature} = `sensors`;
$res->{HDDtemperature} = `hddtemp /dev/sda`;
```

Mở file `/usr/share/pve-manager/js/pvemanagerlib.js`:
- Tìm `height: 300` và sửa thành `height: 420`
- Tìm `pveversion` ở gần dòng 34827, thêm vào bên dưới:

```
        {
            itemId: 'version',
            colspan: 2,
            printBar: false,
            title: gettext('PVE Manager Version'),
            textField: 'pveversion',
            value: ''
        },
#add===
        {
            itemId: 'CPUtemperature',
            colspan: 2,
            printBar: false,
            title: gettext('CPU Temperature'),
            textField: 'CPUtemperature',
            renderer: function(value){
                const dieTemp = Array.from(value.matchAll(/Tdie:.*?\+([\d\.]+)?/g), m=>m[1]);
                return dieTemp.map((element, index) => `CPU Die ${index+1}: ${element}℃`).join(' | ');
                                        }
        },
        {
            itemId: 'GPUtemperature',
            colspan: 2,
            printBar: false,
            title: gettext('GPU Temperature'),
            textField: 'GPUtemperature',
            renderer: function(value) {
                const temp = value.match(/temp1:.*?\+([\d\.]+)?/);
                const gpuTemp = temp !== null && temp.length > 0 ? temp[1] : "Unavailable ";
                return `GPU: ${gpuTemp}℃ `
            }
        },
        {
            itemId: 'Nvmetemperature',
            colspan: 2,
            printBar: false,
            title: gettext('Nvme Temperature'),
            textField: 'Nvmetemperature',
            renderer: function(value){
                const nvmeTemps = Array.from(value.matchAll(/Composite.*?\+([\d\.]+)?/g), m=>m[1]);
return "Nvme: " + nvmeTemps.map((element, index) => `${element}℃`).join(' | ');
                                        }
        },
        {
            itemId: 'HDDtemperature',
            colspan: 2,
            printBar: false,
            title: gettext('HDD Temperature'),
            textField: 'HDDtemperature',
            renderer: function(value) {
                value = value.replace(/Â/g, '');
                return value.replace(/\n/g, '<br>')
            }
        },
#end===
```

## Khởi động lại service

```
systemctl restart pveproxy
```

Xoá cache của trình duyệt và xem thành quả thôi :))