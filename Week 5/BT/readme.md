### A. Mục tiêu

- Viết 01 chương trình C/C++ có sử dụng thư viện cJSON

- Tự tạo 01 thư viện đơn giản của riêng mình và ứng dụng
sử dụng thư viện đó

- Xây dựng chương trình có phụ thuộc vào cả 2 thư viện ở
Bài 1 và Bài 2 vào Buildroot có ràng buộc phụ thuộc.

---

### B. Thực hiện chi tiết

---

**I. Biên dịch ứng dụng với thư viện đã có**

**1. Kích hoạt thư viện cJSON (Bài 1)**

```
Chạy lệnh: make menuconfig
Tìm đường dẫn: Target packages ---> Libraries ---> JSON/XML ---> [*] cJSON.
```

![1](https://github.com/user-attachments/assets/a4bcae39-3161-44e3-b01e-fc38545b29a7)

```
Save và Exit.
Chạy lệnh để Buildroot tải và cài đặt cJSON vào Staging: make cjson
```

**2. Tạo thư mục bài tập và viết mã nguồn**

```
Sử dụng lệnh tạo thư mục: mkdir -p ~/bai_tap_01
Tạo file bằng lệnh: nano ~/bai_tap_01/HelloJSON.c
```

**Nội dung file C:**

```
C
#include <stdio.h>
#include <stdlib.h>
#include <cjson/cJSON.h>

int main() {
    // 1. Giả lập một chuỗi JSON
    const char *json_string = "{\"name\":\"BeagleBone Black\", \"version\":\"nhom7\", \"status\":1}";

    // 2. Parse chuỗi JSON
    cJSON *root = cJSON_Parse(json_string);
    if (root == NULL) return 1;

    // 3. Trích xuất thông tin
    cJSON *name = cJSON_GetObjectItemCaseSensitive(root, "name");
    cJSON *version = cJSON_GetObjectItemCaseSensitive(root, "version");
    cJSON *status = cJSON_GetObjectItemCaseSensitive(root, "status");

    // 4. In kết quả lên Terminal
    printf("--- KET QUA PARSE GOI TIN JSON ---\n");
    if (cJSON_IsString(name)) printf("Device Name: %s\n", name->valuestring);
    if (cJSON_IsString(version)) printf("Version    : %s\n", version->valuestring);
    if (cJSON_IsNumber(status)) printf("Status Code: %d\n", status->valueint);

    // 5. Giải phóng bộ nhớ
    cJSON_Delete(root);
    return 0;
}
```

**3. Biên dịch chéo và đưa xuống BBB**

Tại thư mục chứa file, chạy lệnh biên dịch:

```
Bash
~/buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-gcc \
-I ~/buildroot/output/host/arm-buildroot-linux-gnueabihf/sysroot/usr/include \
~/bai_tap_01/HelloJSON.c -o ~/bai_tap_01/HelloJSON \
-L ~/buildroot/output/host/arm-buildroot-linux-gnueabihf/sysroot/usr/lib \
-lcjson
```

Kết quả nhận được khi chạy trên BBB:

![2](https://github.com/user-attachments/assets/f8908008-b257-4615-bbdc-cb45501aa4cb)

**II. Bài 2: Tự tạo thư viện cá nhân**

**1. Tạo cấu trúc thư mục**

```
Chạy lệnh: mkdir -p package/mylib/src
```

**2. Viết mã nguồn cho thư viện**

```
Tạo file Header bằng lệnh: nano package/mylib/src/mylib.h
```

```
C
#ifndef __MYLIB_H__
#define __MYLIB_H__

int cong(int a, int b);
int tru(int a, int b);

#endif
```

```
Tạo file Source bằng lệnh: nano package/mylib/src/mylib.c
```

```
C
#include "mylib.h"

int cong(int a, int b) {
    return a + b;
}

int tru(int a, int b) {
    return a - b;
}
```

**3. Tạo file cấu hình Config.in**

```
Chạy lệnh: nano package/mylib/Config.in
```

```
Plaintext
config BR2_PACKAGE_MYLIB
	bool "mylib"
	help
	  Thu vien toan hoc don gian cua Duy Manh.
```

**4. Tạo makefile**

```
Chạy lệnh: nano package/mylib/mylib.mk
```

```
Makefile
################################################################################
# mylib
################################################################################
MYLIB_VERSION = 1.0
MYLIB_SITE = $(TOPDIR)/package/mylib/src
MYLIB_SITE_METHOD = local
MYLIB_INSTALL_STAGING = YES

define MYLIB_BUILD_CMDS
	$(TARGET_CC) $(TARGET_CFLAGS) -fPIC -c $(@D)/mylib.c -o $(@D)/mylib.o
	$(TARGET_AR) rcs $(@D)/libmylib.a $(@D)/mylib.o
	$(TARGET_CC) $(TARGET_CFLAGS) -shared -o $(@D)/libmylib.so $(@D)/mylib.o
endef

define MYLIB_INSTALL_STAGING_CMDS
	$(INSTALL) -D -m 0644 $(@D)/mylib.h $(STAGING_DIR)/usr/include/mylib.h
	$(INSTALL) -D -m 0644 $(@D)/libmylib.a $(STAGING_DIR)/usr/lib/libmylib.a
	$(INSTALL) -D -m 0755 $(@D)/libmylib.so $(STAGING_DIR)/usr/lib/libmylib.so
endef

define MYLIB_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/libmylib.so $(TARGET_DIR)/usr/lib/libmylib.so
endef

$(eval $(generic-package))
```

**5. Đăng ký và biên dịch**

```
Mở file đăng ký: nano package/Config.in, thêm dòng source "package/mylib/Config.in":
```

![3](https://github.com/user-attachments/assets/483d0f4f-022c-40f0-a0b9-2843e1ea3c16)

```
Chạy make menuconfig -> Target packages -> chọn mylib:
```

![4](https://github.com/user-attachments/assets/8fd6d609-cc62-431b-8283-2944700edd21)

```
Biên dịch: make mylib
```

**6. Viết chương trình test và so sánh kết quả**

```
Tạo file test: nano ~/bai_tap_02/test_lib.c
```

```
C
#include <stdio.h>
#include <mylib.h>

int main() {
    printf("--- CHUONG TRINH CUA DUY MANH - TEST LIB CONG TRU ---\n");
    int a = 20, b = 10;
    printf("Ket qua: %d + %d = %d\n", a, b, cong(a, b));
    printf("Ket qua: %d - %d = %d\n", a, b, tru(a, b));
    return 0;
}
```

```
Biên dịch 2 bản tĩnh và động, đưa xuống BBB chạy thử:
```

![6](https://github.com/user-attachments/assets/7575cb50-0434-4531-bbef-f08af9d51a9a)


**7. So sánh kích thước và sự phụ thuộc**

```
Kiểm tra bằng lệnh ls -lh và readelf -d:
```

![5](https://github.com/user-attachments/assets/4896f756-e5c6-414e-a6b2-c9de14cc5a80)

```
Nhận xét: File app_dynamic rất nhẹ (7.8K) nhưng phụ thuộc vào libmylib.so. File app_static nặng hơn (507K) nhưng chạy độc lập.
```

---

**III. Bài 3: Tích hợp ứng dụng và thư viện vào Buildroot**

**1. Tạo cấu trúc và mã nguồn**

```
Tạo thư mục: mkdir -p package/myapp/src
Tạo file code: nano package/myapp/src/myapp.c
```

```
C
#include <stdio.h>
#include <stdlib.h>
#include <cjson/cJSON.h>
#include <mylib.h>

int main() {
    printf("--- CHUONG TRINH TICH HOP (cJSON + mylib) CUA DUY MANH ---\n");
    int tong = cong(50, 25);
    int hieu = tru(50, 25);

    cJSON *root = cJSON_CreateObject();
    cJSON_AddStringToObject(root, "Tac_gia", "Duy Manh");
    cJSON_AddNumberToObject(root, "Phep_Cong", tong);
    cJSON_AddNumberToObject(root, "Phep_Tru", hieu);

    char *json_string = cJSON_Print(root);
    printf("Ket qua JSON:\n%s\n", json_string);

    cJSON_Delete(root);
    free(json_string);
    return 0;
}
```

**2. Tạo file cấu hình và makefile**

```
Tạo Config.in: nano package/myapp/Config.in
```

```
Plaintext
config BR2_PACKAGE_MYAPP
	bool "myapp (Duy Manh Integration)"
	select BR2_PACKAGE_CJSON
	select BR2_PACKAGE_MYLIB
	help
	  Chuong trinh tich hop dung ca cJSON (Bai 1) va mylib (Bai 2).
```

```
Tạo makefile: nano package/myapp/myapp.mk
```

```
Makefile
################################################################################
# myapp
################################################################################
MYAPP_VERSION = 1.0
MYAPP_SITE = $(TOPDIR)/package/myapp/src
MYAPP_SITE_METHOD = local
MYAPP_DEPENDENCIES = cjson mylib

define MYAPP_BUILD_CMDS
	$(TARGET_CC) $(TARGET_CFLAGS) $(@D)/myapp.c -o $(@D)/myapp -lcjson -lmylib
endef

define MYAPP_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/myapp $(TARGET_DIR)/usr/bin/myapp
endef

$(eval $(generic-package))
```

**3. Đăng ký, Chọn và Biên dịch**

```
Đăng ký vào package/Config.in:
```

![8](https://github.com/user-attachments/assets/ff8593a6-92ca-44c0-a925-b4930d77eedd)

```
Chạy make menuconfig và chọn myapp:
```

![7](https://github.com/user-attachments/assets/0b1aeecd-6f54-49dc-bb31-de764b335b4c)

```
Biên dịch ứng dụng: make myapp
```

**4. Khởi chạy trên BBB**

```
Sau khi copy file myapp xuống BeagleBone Black, tiến hành chạy ứng dụng.
```

**Kết quả thu được:**

![9](https://github.com/user-attachments/assets/80c33846-85c9-4479-9ef9-eef1d4ba8d6b)
