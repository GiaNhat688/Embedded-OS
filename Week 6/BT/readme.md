### I. GIAO TIẾP VỚI DEVICE DRIVER THỦ CÔNG (CAT/ECHO)

**Mục tiêu: Kích hoạt Driver và kiểm tra khả năng tương tác trực tiếp qua thư mục /dev/.**

---

**1. Giải phóng LED: Mặc định LED được Kernel quản lý, em thực hiện tước quyền để tự điều khiển:**

```
Lệnh: echo none > /sys/class/leds/beaglebone:green:usr0/trigger
```

**2. Tạo Device Node ảo: Để ứng dụng giao tiếp qua /dev/ theo yêu cầu:**

```
Lệnh: ln -sf /sys/class/leds/beaglebone:green:usr0/brightness /dev/gpio_led
```
**3. Điều khiển thủ công:**

```
Bật LED: echo 1 > /dev/gpio_led
Tắt LED: echo 0 > /dev/gpio_led
Đọc trạng thái: cat /dev/gpio_led (Kết quả trả về 1 khi đèn sáng và 0 khi đèn tắt).
```

![1 (1)](https://github.com/user-attachments/assets/b742c604-bb45-434a-b9ee-0a947f744a37)

_(Chú thích: Minh chứng thao tác echo/cat thành công trên Terminal)_

--- 

### II. QUY TRÌNH LẬP TRÌNH ỨNG DỤNG C (BLINK LED)

**Mục tiêu: Tạo ứng dụng tự động hóa việc nháy LED.**

---

**1. Tạo thư mục làm việc:**

```
Lệnh: mkdir -p ~/buildroot/package/blinkled
```

**2. Tạo mã nguồn blink.c:**

```
Lệnh: nano ~/buildroot/package/blinkled/blink.c
```

**Nội dung code:**

```
C
#include <stdio.h>
#include <unistd.h>

int main() {
    FILE *fp;
    printf("Chuong trinh Blink LED dang khoi chay...\n");

    while(1) {
        fp = fopen("/dev/gpio_led", "w");
        if (fp != NULL) {
            fprintf(fp, "1"); // Bật LED
            fflush(fp);
            fclose(fp);
        }
        sleep(1); // Trễ 1 giây

        fp = fopen("/dev/gpio_led", "w");
        if (fp != NULL) {
            fprintf(fp, "0"); // Tắt LED
            fflush(fp);
            fclose(fp);
        }
        sleep(1); // Trễ 1 giây
    }
    return 0;
}
```

---

### III. ĐÓNG GÓI VÀO BUILDROOT (MAKEFILE & CONFIG)

**Mục tiêu: Sử dụng cơ chế biên dịch chéo để tích hợp ứng dụng vào Image hệ điều hành.**

---

**1. Tạo file cấu hình Config.in:**

```
Lệnh: nano ~/buildroot/package/blinkled/Config.in
```

**Nội dung:**

```
Plaintext
config BR2_PACKAGE_BLINKLED
    bool "blinkled"
    help
      Ung dung tu dong nhay LED cho BeagleBone Black (Duy Manh).
```

**2. Tạo file biên dịch blinkled.mk:**

```
Lệnh: nano ~/buildroot/package/blinkled/blinkled.mk
```

**Nội dung:**

```
Makefile
BLINKLED_VERSION = 1.0
BLINKLED_SITE = $(TOPDIR)/package/blinkled
BLINKLED_SITE_METHOD = local

# Lệnh biên dịch file C cho chip ARM
define BLINKLED_BUILD_CMDS
	$(TARGET_CC) $(TARGET_CFLAGS) $(@D)/blink.c -o $(@D)/blinkled
endef

# Lệnh cài đặt file thực thi và script vào hệ thống
define BLINKLED_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/blinkled $(TARGET_DIR)/usr/bin/blinkled
	$(INSTALL) -D -m 0755 $(@D)/S99blinkled $(TARGET_DIR)/etc/init.d/S99blinkled
endef

$(eval $(generic-package))
```

---

### IV. CẤU HÌNH TỰ KHỞI CHẠY (AUTO-RUN)

**Mục tiêu: Ứng dụng tự chạy ngay sau khi OS khởi động thành công.**

---

**1. Tạo Script khởi động:**

```
Lệnh: nano ~/buildroot/package/blinkled/S99blinkled
```

**Nội dung:**

```
Bash
#!/bin/sh
case "$1" in
  start)
    echo "He thong dang tu dong kich hoat LED..."
    echo none > /sys/class/leds/beaglebone:green:usr0/trigger
    ln -sf /sys/class/leds/beaglebone:green:usr0/brightness /dev/gpio_led
    /usr/bin/blinkled &
    ;;
  stop)
    killall blinkled
    ;;
esac
```

**2. Cấp quyền thực thi:**

```
Lệnh: chmod +x ~/buildroot/package/blinkled/S99blinkled
```

**3. Kích hoạt trong Menuconfig:**

```
Gõ lệnh: make menuconfig
Thao tác: Chọn Target packages -> Tìm và tích chọn [*] blinkled.
```

**4. Biên dịch toàn bộ hệ thống:**

```
Gõ lệnh: make
```

---

### V. NẠP THẺ NHỚ VÀ VẬN HÀNH THỰC TẾ

**Mục tiêu: Triển khai Image lên phần cứng và kiểm tra kết quả.**

---

**1. Nạp Image vào thẻ nhớ:**

```
Lệnh xác định thẻ: lsblk
Lệnh nạp: sudo dd if=output/images/sdcard.img of=/dev/sdb bs=4M status=progress
Chốt dữ liệu: sync
```

**2. Kiểm tra trên mạch BBB:**

```
Cắm thẻ và cấp nguồn: LED USR0 tự động nhấp nháy sau 20 giây.
Kiểm tra tiến trình ngầm: ps | grep blinkled
```

![4 (1)](https://github.com/user-attachments/assets/70075a48-e25d-49a9-9e8d-3fa92d6c4d7b)

_(Chú thích: Kết quả lệnh ps trên mạch BBB xác nhận ứng dụng tự chạy thành công)_

---

**VI. KẾT LUẬN**

```
Hoàn thành đầy đủ các yêu cầu từ tương tác Driver thủ công đến đóng gói ứng dụng.
Hệ thống tự khởi chạy ổn định, minh chứng cho việc làm chủ quy trình Buildroot và cơ chế Init Script của Linux Nhúng.
```
