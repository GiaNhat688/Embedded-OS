# A. Mục tiêu

- Thực hiện build một hệ điều hành hoàn chỉnh bằng công cụ Buildroot.

- Tự động hóa quá trình tạo các file phân vùng boot gồm: MLO, u-boot.img, zImage, am335x-boneblack.dtb và rootfs.

- Sử dụng Toolchain để viết chương trình C.

- Biên dịch chương trình bằng Toolchain đã build.

- Chạy thành công chương trình C trên BeagleBone Black (BBB).

---

# B. Build hệ điều hành

**1. Chuẩn bị môi trường (Host Setup)**

Em thực hiện cài đặt các gói thư viện bổ trợ trên Ubuntu bằng lệnh:

```
Bash
sudo apt update
sudo apt install build-essential checkinstall libncursesw5-dev \
python3-dev python3-setuptools-pynacl python3-pip \
libssl-dev curltcl-dev git bc bzr cvs mercurial \
unzip wget rsync fastjar java-wrappers bison flex texinfo -y
```

**2. Tải mã nguồn Buildroot**

Bash
git clone https://github.com/buildroot/buildroot.git
cd buildroot
git checkout 2023.11.x

**3. Cấu hình cho BeagleBone Black**

Áp dụng cấu hình mặc định (defconfig) cho board BBB:

```
Bash
make beaglebone_defconfig
```

**4. Cấu hình hệ thống (menuconfig)**

- Chạy lệnh make menuconfig và thiết lập các thông số:

  - Target Options: Target Architecture (ARM), Variant (cortex-A8), ABI (EABIhf).

  - Toolchain: C library (glibc).

  - Kernel: Latest CIP SLTS version (5.10.162-cip24), Defconfig name (omap2plus), Device Tree (am335x-boneblack).

  - System Configuration: Run a getty (ttyS0 - 115200).

**Tiến hành Build: Sử dụng lệnh make -j$(nproc).**

<img width="729" height="449" alt="image" src="https://github.com/user-attachments/assets/1da89254-55db-456b-ae7d-aeb8142a1ead" />

_(Chú thích: Các file output thu được trong thư mục output/images)_

**5. Copy vào thẻ nhớ**

Sử dụng file sdcard.img để nạp vào thẻ nhớ (thiết bị /dev/sdb):

```
Bash
sudo dd if=output/images/sdcard.img of=/dev/sdb bs=4M status=progress
sync
```

Chỉnh sửa nội dung file /media/manhht10ht/BOOT/extlinux/extlinux.conf:

```
Plaintext
label beaglebone-buildroot
  kernel /zImage
  fdt /am335x-boneblack.dtb
  append console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait
```

Kết quả thu được sau khi boot trên BBB:

<img width="729" height="449" alt="image" src="https://github.com/user-attachments/assets/befdaaa6-914a-4892-a736-2eb86cddbeb9" />

_(Chú thích: Cấu trúc thư mục RootFS và các lệnh trong /bin trên hệ điều hành mới)_

---

# C. Sử dụng toolchain từ buildroot

**1. Tạo mã nguồn C trên máy host**

Em tạo thư mục làm việc và soạn thảo file hello_manh.c:

```
Bash
mkdir ~/bai_tap_02
cd ~/bai_tap_02
nano hello_manh.c
```

Nội dung mã nguồn:

```
C
#include <stdio.h>

int main(void) {
    printf("--------------------------------------\n");
    printf(" Xin chao, day la chuong trinh cua Duy Manh.\n");
    printf(" Chay tren BeagleBone Black - OS Buildroot\n");
    printf("--------------------------------------\n");
    return 0;
}
```

---

**2. Sử dụng toolchain của buildroot để biên dịch**

Đường dẫn trình biên dịch chéo: ~/buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-gcc Sử dụng câu lệnh:

```
Bash
~/buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-gcc -o hello_manh hello_manh.c
```

**3. Đưa chương trình vào RootFS trên thẻ nhớ**

Gắn thẻ nhớ vào Ubuntu và copy file thực thi vào thư mục /usr/bin:

```
Bash
sudo cp hello_manh /media/manhht10ht/ROOTFS/usr/bin/
sudo chmod +x /media/manhht10ht/ROOTFS/usr/bin/hello_manh
```

**4. Chạy thử lệnh trên BBB**

Thực hiện đăng nhập quyền root và chạy lệnh trực tiếp trên Terminal của mạch:

```
Bash
hello_manh
```

<img width="738" height="154" alt="image" src="https://github.com/user-attachments/assets/dbd9ff59-04e8-45f0-a57b-225acfca16ac" />

_(Chú thích: Chương trình hello_manh thực thi thành công trên BeagleBone Black)_
