weston fbdev-backend by VNC server
===
### 本文
由於wayvnc不支援太舊的weston版本?
![](https://i.imgur.com/hnMOf9y.png)

本文章使用framebuffer backend實作遠端桌面(remote Desktop)

使用的wayland compositor:
>weston 5.0.0
>需要/usr/lib/libweston-5/fbdev-backend.so

library :
>libvncserver
>framebuffer-vncserver

先透過強行啟動HDMI裝置分配framebuffer，再使用fbdev-backend 將螢幕繪製到framebuffer。
然後使用uinput創造出虛擬的鍵盤滑鼠供fbdev-backend和framebuffer-vncserver使用。
最後只要用VNC Viewer就能連上畫面了
### libvncserver
1. Download
>git clone https://github.com/LibVNC/libvncserver.git

2. set cross compiler env
>source /opt/fsl-imx-wayland/4.14-sumo/environment-setup-aarch64-poky-linux

3. install
>cmake CMakeList.txt
>
>make  DESTDIR=your_dest install

### framebuffer-vncserver
1. Download
>git clone https://github.com/ponty/framebuffer-vncserver.git
2. set cross compiler env
> source /opt/fsl-imx-wayland/4.14-sumo/environment-setup-aarch64-poky-linux
3. 修改src/touch.c
由於weston的觸發使用滑鼠左鍵，修改BTN_TOUCH > BTN_LEFT
第205行:
```javascript=200
        // Then send a BTN_TOUCH
        gettimeofday(&time, 0);
        ev.input_event_sec = time.tv_sec;
        ev.input_event_usec = time.tv_usec;
        ev.type = EV_KEY;
        ev.code = BTN_TOUCH; => ev.code = BTN_LEFT;  
        ev.value = touchValue;
```
3. Install
>cmake CMakeList.txt
>make  DESTDIR=your_dest install


※ Dependency:
>libvncserver

### uinput
使用uinput創出虛擬的滑鼠和鍵盤(/dev/input/event*)
程式碼如下

```javascript=
#include <linux/uinput.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
    
using namespace std;
void mouse_move(int fd, int dx, int dy)
{
    struct input_event ev;

    memset(&ev, 0, sizeof(struct input_event));
    ev.type = EV_ABS;
    ev.code = ABS_X;
    ev.value = dx;
    if (write(fd, &ev, sizeof(struct input_event)) < 0) {
        printf("move error\n");
    }

    memset(&ev, 0, sizeof(struct input_event));
    ev.type = EV_ABS;
    ev.code = ABS_Y;
    ev.value = dy;
    if (write(fd, &ev, sizeof(struct input_event)) < 0) {
        printf("move error\n");
    }


    memset(&ev, 0, sizeof(struct input_event));
    ev.type = EV_SYN;
    ev.code = SYN_REPORT;
    ev.value = 0;
    if (write(fd, &ev, sizeof(struct input_event)) < 0) {
        printf("move error\n");
    }
}

void mouse_report_key(int fd, uint16_t type, uint16_t keycode, int32_t value)
{
    struct input_event ev;

    memset(&ev, 0, sizeof(struct input_event));

    ev.type = type;
    ev.code = keycode;
    ev.value = value;

    if (write(fd, &ev, sizeof(struct input_event)) < 0) {
        printf("key report error\n");
    }else{
        printf("refrush,%d,%d,%d\n",type,keycode,value);
    }
}

int main(void)
{

    struct uinput_user_dev mouse,key;
    int fd,fd2, ret ,version;

    int dx, dy;

    fd = open("/dev/uinput", O_WRONLY | O_NONBLOCK);
    fd2 = open("/dev/uinput", O_WRONLY | O_NONBLOCK);
    if (fd < 0 && fd2 < 0) {
        return fd;
    }
    //key
    ioctl(fd2, UI_SET_EVBIT, EV_KEY);
    for(int i=0;i<=120;i++)
        ioctl(fd2, UI_SET_KEYBIT, i);
    //ioctl(fd2, UI_SET_KEYBIT, 133);
    memset(&key, 0, sizeof(struct uinput_user_dev));
    snprintf(key.name, UINPUT_MAX_NAME_SIZE, "key");
    key.id.bustype = BUS_USB;
    key.id.vendor = 0x1234;
    key.id.product = 0x5679;
    key.id.version = 1;

    ret = write(fd2, &key, sizeof(struct uinput_user_dev));
    ioctl(fd2, UI_DEV_SETUP, &key);
    ret = ioctl(fd2, UI_DEV_CREATE);
    if (ret < 0) {
        close(fd2);
        return ret;
    }
    //mouse
    ioctl(fd, UI_SET_EVBIT, EV_SYN);

    ioctl(fd, UI_SET_EVBIT, EV_KEY);
    ioctl(fd, UI_SET_KEYBIT, BTN_TOUCH);
    ioctl(fd, UI_SET_KEYBIT, BTN_MOUSE);
    ioctl(fd, UI_SET_KEYBIT, BTN_LEFT);
    ioctl(fd, UI_SET_KEYBIT, BTN_RIGHT);
    ioctl(fd, UI_SET_KEYBIT, BTN_MIDDLE);

    ioctl(fd, UI_SET_EVBIT, EV_REL);
//    ioctl(fd, UI_SET_RELBIT, REL_X);
//    ioctl(fd, UI_SET_RELBIT, REL_Y);

    ioctl(fd, UI_SET_EVBIT, EV_ABS);
    ioctl(fd, UI_SET_ABSBIT, ABS_X);
    ioctl(fd, UI_SET_ABSBIT, ABS_Y);
//    ioctl(fd, UI_SET_ABSBIT, ABS_PRESSURE);
//    ioctl(fd, UI_SET_ABSBIT, ABS_MT_POSITION_X);
//    ioctl(fd, UI_SET_ABSBIT, ABS_MT_POSITION_Y);

    memset(&mouse, 0, sizeof(struct uinput_user_dev));
    snprintf(mouse.name, UINPUT_MAX_NAME_SIZE, "mouse");
    mouse.id.bustype = BUS_USB;
    mouse.id.vendor = 0x1234;
    mouse.id.product = 0x5678;
    mouse.id.version = 1;

    mouse.absmin[ABS_X] = 0;
    mouse.absmax[ABS_X] = 1023;
    mouse.absfuzz[ABS_X] = 0;
    mouse.absflat[ABS_X] = 0;

    mouse.absmin[ABS_Y] = 0;
    mouse.absmax[ABS_Y] = 600;
    mouse.absfuzz[ABS_Y] = 0;
    mouse.absflat[ABS_Y] = 0;

    ret = write(fd, &mouse, sizeof(struct uinput_user_dev));
    ioctl(fd, UI_DEV_SETUP, &mouse);
    ret = ioctl(fd, UI_DEV_CREATE);
    if (ret < 0) {
        close(fd);
        return ret;
    }


//    sleep(1);
//    mouse_move(fd, 10, 10);
//    mouse_move(fd, 20, 20);

    mouse_report_key(fd, EV_KEY, BTN_RIGHT, 1);
    mouse_report_key(fd, EV_SYN, SYN_REPORT, 0);
    mouse_report_key(fd, EV_KEY, BTN_RIGHT, 0);
    mouse_report_key(fd, EV_SYN, SYN_REPORT, 0);

    dx = dy = 10;
    while (1) {
        //mouse_move(fd, dx, dy);
        sleep(1);
    }

    ioctl(fd, UI_DEV_DESTROY);
    ioctl(fd2, UI_DEV_DESTROY);

    close(fd);
    close(fd2);

    return 0;
}
```
reference:
>https://www.kernel.org/doc/html/v4.12/input/uinput.html
>https://blog.csdn.net/mcgrady_tracy/article/details/28340931
### weston 
我啟動weston時的配置文件
Path:/etc/xdg/weston/weston.ini
```javascript=
[core]
gbm-format=argb8888
idle-time=0
backend=fbdev-backend.so
require-input=true
use-pixman=true

[shell]
background-image=/home/desktop.png
size=1024x600


[libinput]
enable-tap=true
tap-and-drag=true
nd-drag-lock=true
touchscreen_calibrator=true
[launcher]
#path=

[input-method]
path=/usr/libexec/weston-keyboard

[output]
name=HDMI-A-1
transform=90
mode=current

[screen-share]
command=/usr/bin/weston  --backend=fbdev-backend.so --shell=fullscreen-shell.so --no-clients-resize 
```
reference:
>http://manpages.ubuntu.com/manpages/focal/man5/weston.ini.5.html
>※找不到5.0.0版本的weston.ini說明

### environment
1.啟動framebuffer裝置(/dev/fb0)
為了產生framebuffer
在開機的時候不使用實體HDMI，強制啟動HDMI設備

在開機檔案 /etc/rc.local 中加上:

>echo on > /sys/devices/platform/display-subsystem/drm/card0/card0-HDMI-A-1/status

※ reference:
>https://markyzq.gitbooks.io/rockchip_drm_integration_helper/content/zh/drm_force_enable.html

2.為了讓framebuffer-vncserver可以使用虛擬的鍵盤滑鼠(virtual keyboard and mouse)，使用uinput新增裝置(/dev/input/event*)

### 執行
啟動你的虛擬裝置
$./your_uinput
啟動weston螢幕
$weston --tty1
啟動VNCserver
$framebuffer-vncserver -f /dev/fb0 -k /dev/input/event1 -t /dev/input/event2  -F 16
※-f framebuffer螢幕，-k keyboard鍵盤，-t touch滑鼠，-F fps 目前只支援到16
※裝置根據啟動weston讀取到的裝置為主
![](https://i.imgur.com/fXdItOG.png)


最後只要使用vnc Viewer就可以連上server了![](https://i.imgur.com/nUhIXXQ.gif)
