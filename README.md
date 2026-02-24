# ** TUẦN 3: BUILD ROOT FILESYSTEM VỚI BUSYBOX**


---

**A. MỤC TIÊU**

- Biên dịch thành công RootFS sử dụng BusyBox

- Đưa được các file đã biên dịch từ BusyBox vào thẻ nhớ

- Liên kết Kernel với RootFS thông qua lệnh root= trong bootargs

- Khởi động thành công RootFS. Có thể gõ một số lệnh như: ls, cat, echo...

--

**B. THỰC HIỆN**

---

**BÀI 1: BIÊN DỊCH ROOTFS VỚI BUSYBOX**

---

**Bước 1: Tải và giải nén BusyBox**

```
wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2

tar -xjf busybox-1.36.1.tar.bz2

cd busybox-1.36.1

```

**Bước 2: Cấu hình**

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- defconfig
make menuconfig
```

**Bước 3: Biên dịch**

```
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```

**Bước 4: Cài vào thư mục rootfs**

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- CONFIG_PREFIX=../rootfs install
```
---

**BÀI 2: TẠO CẤU TRÚC ROOTFS CHUẨN **

---

**Bước 1: Vào rootfs**

```
cd ../rootfs
mkdir -p {dev,etc,proc,sys,tmp,var,home}
```

**Bước 2: Tạo file init**

```
vi init
```

**Bước 3: Nội dung**

```
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "Hello RootFS"
exec /bin/sh
```

**Bước 4: Cấp quyền**

```
chmod +x init
```
---

**BÀI 3: ĐƯA FILE BUSYBOX VÀO THẺ NHỚ**

```
sudo cp -r rootfs/* /media/sdcard/
sync
```

---

**BÀI 4: LIÊN KẾT KERNEL VỚI ROOTFS QUA BOOTARGS**

---

```
setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p2 rw init=/init
saveenv
boot
```

---

**KHỞI ĐỘNG THÀNH CÔNG**

```
Hello RootFS
/ #
```

---

**THỬ LỆNH**

```
ls
cat /proc/cpuinfo
echo hi
```
