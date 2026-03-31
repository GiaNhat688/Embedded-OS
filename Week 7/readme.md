A. MỤC TIÊU:

- Hoàn thiên 1 Driver có đủ các hàm cơ bản bao gồm:
  
  - Init/Exit
    
  - Open/Release
    
  - Read/Write
    
  - Đăng ký thông qua static struct file_operations
    
- Tự động cấp phát Major/Minor
  
- Tự động tạo file Device trong thư mục /dev/ theo tên chỉ định
  
  - Tạo Class
    
  - Tạo Device
    
  - Liên kết Device file với Major/Minor tương ứng
    
- Có khả năng đọc dữ liệu từ User Space và ghi dữ liệu vào User Space
  
  - Sử dụng hàm copy_from_user
    
  - Sử dụng hàm copy_to_user
 
- Mở rộng driver trên cho phép tương tác ngoại vi GPIO

  - Sử dụng ioremap để truy cập bộ nhớ vật lý.

  - Cấu hình chân LED ở chế độ phù hợp, tham khảo Reference Manual.
  
  - Cung cấp lệnh đọc trạng thái và ghi trạng thái LED thông qua Device File đã khai báo ở trên
    
---
 
**B. THỰC HIỆN:**

**PHẦN 1.1: LẬP TRÌNH VÀ BIÊN DỊCH TRÊN UBUNTU HOST**

**Bước 1: Tạo không gian làm việc Mở Terminal trên Ubuntu và tạo thư mục dự án mới:**

```
Bash

cd ~

mkdir -p ~/driver_manh_final

cd ~/driver_manh_final
```

**Bước 2: Viết mã nguồn C cho Driver Sử dụng trình soạn thảo nano để viết code:**

```
Bash
nano manh_driver.c
```

Nội dung file manh_driver.c được thiết kế với cơ chế tự động hóa và chống tràn bộ nhớ (Buffer Overflow):

```
C
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "manh_device"
#define CLASS_NAME  "manh_class"
#define BUF_SIZE    1024

static dev_t dev_num;
static struct cdev my_cdev;
static struct class *my_class;
static char kernel_buffer[BUF_SIZE];

/* Hàm Open & Release quản lý vòng đời truy cập thiết bị */
static int my_open(struct inode *inode, struct file *file) {
    pr_info("Driver Duy Manh: Da Open() thiet bi!\n");
    return 0;
}

static int my_release(struct inode *inode, struct file *file) {
    pr_info("Driver Duy Manh: Da Release() thiet bi!\n");
    return 0;
}

/* Hàm Read: Gửi dữ liệu từ Kernel lên User Space */
static ssize_t my_read(struct file *file, char __user *user_buf, size_t size, loff_t *offset) {
    size_t datalen = strlen(kernel_buffer);
    if (*offset >= datalen) return 0; // Kiem tra EOF
    if (size > datalen - *offset) size = datalen - *offset;

    if (copy_to_user(user_buf, kernel_buffer + *offset, size)) return -EFAULT;
    *offset += size;
    pr_info("Driver Duy Manh: Da gui %zu bytes len User Space\n", size);
    return size;
}

/* Hàm Write: Nhận dữ liệu từ User Space xuống Kernel */
static ssize_t my_write(struct file *file, const char __user *user_buf, size_t size, loff_t *offset) {
    if (size > BUF_SIZE - 1) size = BUF_SIZE - 1; // Chong Buffer Overflow
    if (copy_from_user(kernel_buffer, user_buf, size)) return -EFAULT;
    
    kernel_buffer[size] = '\0';
    pr_info("Driver Duy Manh: Da nhan %zu bytes tu User Space\n", size);
    return size;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .release = my_release,
    .read = my_read,
    .write = my_write,
};

/* Hàm Init: Khởi tạo và cấp phát tự động */
static int __init manh_init(void) {
    alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    cdev_init(&my_cdev, &fops);
    cdev_add(&my_cdev, dev_num, 1);
    my_class = class_create(THIS_MODULE, CLASS_NAME);
    device_create(my_class, NULL, dev_num, NULL, DEVICE_NAME);
    pr_info("Driver Duy Manh khoi tao thanh cong! File /dev/%s da san sang.\n", DEVICE_NAME);
    return 0;
}

/* Hàm Exit: Dọn dẹp tài nguyên */
static void __exit manh_exit(void) {
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    pr_info("Driver Duy Manh: Da go bo.\n");
}

module_init(manh_init);
module_exit(manh_exit);
MODULE_LICENSE("GPL");
```

**Bước 3: Viết Makefile và Biên dịch Tạo file Makefile để phục vụ biên dịch chéo:**

```
Bash
nano Makefile
```

**Nội dung file Makefile:**

```
Makefile
obj-m += manh_driver.o

KDIR := /home/manhht10ht/buildroot/output/build/linux-custom
CROSS := /home/manhht10ht/buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-

all:
	make -C $(KDIR) M=$(PWD) ARCH=arm CROSS_COMPILE=$(CROSS) modules

clean:
	make -C $(KDIR) M=$(PWD) clean

Thực hiện lệnh biên dịch và copy file .ko sinh ra vào thẻ nhớ:
Bash
make
sudo mount /dev/sdb2 /media/manhht10ht/rootfs
sudo cp manh_driver.ko /media/manhht10ht/rootfs/root/
sync
sudo umount /media/manhht10ht/rootfs
```

---

**PHẦN 1.2: THỰC THI VÀ KIỂM THỬ TRÊN BEAGLEBONE BLACK**

Cắm thẻ nhớ chứa hệ điều hành vào mạch BeagleBone, cấp nguồn và mở Terminal kết nối qua cổng Serial:

```
Bash
sudo picocom -b 115200 /dev/ttyUSB0
```

_(Đăng nhập vào hệ thống bằng quyền root)_

**Bước 1: Nạp Driver và kiểm tra khởi tạo**

```
Bash
insmod /root/manh_driver.ko
dmesg | tail -n 2
```

![1](https://github.com/user-attachments/assets/b96a4d50-8d92-4177-9d0f-a94ebdc050e6)

_Dòng log Driver Duy Manh khoi tao thanh cong! chứng minh hàm manh_init đã được Kernel gọi. Hệ thống đã thực hiện cấp phát số định danh và hoàn tất đăng ký Driver thành công._

**Bước 2: Kiểm tra tính năng Tự động tạo Device Node**

```
Bash
ls -l /dev/manh_device
```

![2](https://github.com/user-attachments/assets/9a86192d-92bc-472b-99c2-6e39995c1e7a)

_Mặc dù không sử dụng lệnh mknod, file thiết bị /dev/manh_device vẫn tự động xuất hiện với quyền crw-rw----. Hệ thống đã cấp phát số Major tự động là 246. Điều này chứng tỏ hai hàm class_create và device_create đã hoạt động hoàn hảo_

**Bước 3: Kiểm thử truyền dữ liệu User -> Kernel (Hàm Write)**

```
Bash
echo "Duy Manh PTIT" > /dev/manh_device
dmesg | tail -n 3
```

![3](https://github.com/user-attachments/assets/03f97d2b-73d1-40ba-9009-d05e566b57d3)

_Log hiển thị quá trình Open() -> Da nhan 14 bytes tu User Space -> Release(). Lệnh echo đã đẩy thành công chuỗi ký tự "Duy Manh PTIT" xuống nhân hệ điều hành thông qua cơ chế an toàn copy_from_user._

**Bước 4: Kiểm thử đọc dữ liệu Kernel -> User (Hàm Read)**

```
Bash
cat /dev/manh_device
dmesg | tail -n 5
```

![4](https://github.com/user-attachments/assets/25bcbaea-3e74-4dc0-a32f-4f04b8aef922)

_Lệnh cat hiển thị chính xác chuỗi Duy Manh PTIT ra màn hình Terminal. Log xác nhận Da gui 14 bytes len User Space. Driver đã lấy đúng dữ liệu trong Kernel Buffer và trả về cho ứng dụng người dùng thông qua copy_to_user mà không gây ra lỗi tràn bộ nhớ._

---

**PHẦN 1.3: LẬP TRÌNH VÀ BIÊN DỊCH TRÊN UBUNTU HOST**

**Bước 1: Tạo không gian làm việc**

```
Bash
cd ~
mkdir -p ~/driver_gpio_manh
cd ~/driver_gpio_manh
```

**Bước 2: Viết mã nguồn Driver (manh_gpio.c)**

```
Bash
nano manh_gpio.c
```

**Nội dung mã nguồn Driver:**

```
C
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <asm/io.h>

#define DEVICE_NAME "manh_device"
#define CLASS_NAME  "manh_class"
#define GPIO1_BASE  0x4804C000
#define GPIO_SIZE   0x1000
#define GPIO_OE           0x134   // Thanh ghi cấu hình In/Out
#define GPIO_DATAOUT      0x13C   // Thanh ghi đọc trạng thái
#define GPIO_CLEARDATAOUT 0x190   // Thanh ghi Tắt LED
#define GPIO_SETDATAOUT   0x194   // Thanh ghi Bật LED
#define LED_PIN           24      // LED USR3

static dev_t dev_num;
static struct cdev my_cdev;
static struct class *my_class;
void __iomem *gpio_base; 

static int my_open(struct inode *inode, struct file *file) {
    pr_info("Manh GPIO: Da mo file thiet bi!\n");
    return 0;
}

static ssize_t my_read(struct file *file, char __user *user_buf, size_t size, loff_t *offset) {
    char status[2] = "0";
    if (*offset > 0) return 0;
    if (readl(gpio_base + GPIO_DATAOUT) & (1 << LED_PIN)) status[0] = '1';
    if (copy_to_user(user_buf, status, 1)) return -EFAULT;
    pr_info("Manh GPIO: Trang thai LED hien tai la %c\n", status[0]);
    *offset += 1;
    return 1;
}

static ssize_t my_write(struct file *file, const char __user *user_buf, size_t size, loff_t *offset) {
    char cmd;
    if (copy_from_user(&cmd, user_buf, 1)) return -EFAULT;
    if (cmd == '1') writel(1 << LED_PIN, gpio_base + GPIO_SETDATAOUT);
    else if (cmd == '0') writel(1 << LED_PIN, gpio_base + GPIO_CLEARDATAOUT);
    return size;
}

static struct file_operations fops = {.owner = THIS_MODULE, .open = my_open, .read = my_read, .write = my_write};

static int __init manh_gpio_init(void) {
    alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    cdev_init(&my_cdev, &fops);
    cdev_add(&my_cdev, dev_num, 1);
    my_class = class_create(THIS_MODULE, CLASS_NAME);
    device_create(my_class, NULL, dev_num, NULL, DEVICE_NAME);
    gpio_base = ioremap(GPIO1_BASE, GPIO_SIZE);
    writel(readl(gpio_base + GPIO_OE) & ~(1 << LED_PIN), gpio_base + GPIO_OE); // Set Output
    pr_info("Manh GPIO: Khoi tao thanh cong!\n");
    return 0;
}

static void __exit manh_gpio_exit(void) {
    iounmap(gpio_base);
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
    pr_info("Manh GPIO: Da go bo Driver.\n");
}

module_init(manh_gpio_init);
module_exit(manh_gpio_exit);
MODULE_LICENSE("GPL");
```

**Bước 3: Viết mã nguồn App (manh_app.c)**

```
Bash
nano manh_app.c
```

**Nội dung mã nguồn App:**

```
C
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

void blink(int fd, int times, int delay_us) {
    for (int i = 0; i < times; i++) {
        write(fd, "1", 1); printf("LED ON\n"); usleep(delay_us);
        write(fd, "0", 1); printf("LED OFF\n"); usleep(delay_us);
    }
}

int main() {
    int fd = open("/dev/manh_device", O_RDWR);
    if (fd < 0) return -1;
    blink(fd, 5, 500000); // Blink cham
    blink(fd, 10, 100000); // Blink nhanh
    close(fd);
    return 0;
}
```

**Bước 4: Tạo Makefile**

```
obj-m += manh_gpio.o

KDIR := /home/manhht10ht/buildroot/output/build/linux-custom
CROSS := /home/manhht10ht/buildroot/output/host/bin/arm-buildroot-linux-gnueabihf-

all:
	make -C $(KDIR) M=$(PWD) ARCH=arm CROSS_COMPILE=$(CROSS) modules
	$(CROSS)gcc -o manh_app manh_app.c

clean:
	make -C $(KDIR) M=$(PWD) clean
	rm -f manh_app
```

**Bước 5: Biên dịch và Chép vào thẻ nhớ**

```
Bash
make
sudo mount /dev/sdb2 /media/manhht10ht/rootfs
sudo cp manh_gpio.ko manh_app /media/manhht10ht/rootfs/root/
sync && sudo umount /media/manhht10ht/rootfs
```

---

**PHẦN 1.4: THỰC THI VÀ KIỂM THỬ TRÊN BEAGLEBONE BLACK**

_Mở giao tiếp Serial: sudo picocom -b 115200 /dev/ttyUSB0_

**Bước 1: Nạp Driver**

```
Bash
echo none > /sys/class/leds/beaglebone:green:usr3/trigger
insmod /root/manh_gpio.ko
dmesg | tail -n 2
```

![5](https://github.com/user-attachments/assets/eaa0aa6f-f0cb-4751-8929-725fa1b19967)

**Bước 2: Tương tác thủ công (Read/Write)**

```
Bash
echo "1" > /dev/manh_device
cat /dev/manh_device
echo "0" > /dev/manh_device
```

![7](https://github.com/user-attachments/assets/6d12952e-455f-43ff-ae14-8a6e7b94673f)

**Bước 3: Chạy App điều khiển LED**

```
Bash
chmod +x /root/manh_app
/root/manh_app
```

![6](https://github.com/user-attachments/assets/1b3ebcec-1f7c-45c5-82cf-57495df9b844)

**Bước 4: Giải phóng Driver**

```
Bash
rmmod manh_gpio
dmesg | tail -n 2
```

![8](https://github.com/user-attachments/assets/d0596b13-f7f7-4a42-9866-2190bd073066)

---

**C. KẾT LUẬN:**

- Xây dựng thành công Character Driver: Hoàn thiện đầy đủ các hàm cơ bản và cơ chế trao đổi dữ liệu an toàn giữa User Space và Kernel Space.

- Làm chủ ngoại vi qua ioremap: Truy cập trực tiếp bộ nhớ vật lý để cấu hình và điều khiển chính xác các chân GPIO trên BeagleBone Black.

- Kiểm chứng thực tế: Driver vận hành ổn định, cho phép chương trình C lớp người dùng điều khiển Blink LED với các tần số khác nhau.
