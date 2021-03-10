---
title: "GPIO Character Device"
layout: post
comments: true
date: 2021-03-11 14:24
image:
headerImage:
tag:
- Linux
- GPIO
star: false
category: Linux
author: jihun
---

<!-- more -->

# Overview

The GPIO character device has been implemented since kernel 4.8. It will be replaces *sysfs*(`CONFIG_GPIO_SYSFS`) gpio interface.

> The user interface using the ABI based on the sysfs will be deprecated.

In additional, Discovery mechanism(It doesn't use magic numbers) is applied, and gpio pin control is possible through `ioctl()` (`/dev/gpiochip#`).


# Usage in User Application

## Getting Information

### `gpiochip` information

*name*, *label*, and *line number* of gpiochip  are retrieved and shown.

```cpp
int chip_info(int fd)
{
	struct gpiochip_info cinfo;
	int ret = ioctl(fd, GPIO_GET_CHIPINFO_IOCTL, &cinfo);

	if (ret < 0) {
		printf("GPIO_GET_CHIPINFO_IOCTL failed.\n");
		return -1;
	}

	printf("GPIO chip: %s, \"%s\", %u GPIO lines\n",
		cinfo.name, cinfo.label, cinfo.lines);

	return 0;
}
```

### GPIO line information

Retrieves the line name specified by the requested gpio number.
```cpp
int line_info(int fd, int gpio)
{
	int ret;
	struct gpioline_info linfo;

	memset(&linfo, 0, sizeof(linfo));
	linfo.line_offset = gpio;

	ret = ioctl(fd, GPIO_GET_LINEINFO_IOCTL, &linfo);
	if (ret < 0) {
			printf("GPIO_GET_LINEINFO_IOCTL failed.\n");
			return -errno;
	}

	printf("line %2d: %s\n", linfo.line_offset, linfo.name);

	return fd;
}
```

## Access

### Request to use GPIO line

Set the line corresponding to the requested gpiochip number to the mode corresponding to the flag value. You can also specify a label name.

Flags
- GPIOHANDLE_REQUEST_INPUT
- GPIOHANDLE_REQUEST_OUTPUT

```cpp
int request_gpio(int fd, int gpio, int flags, const char *label, int *req_fd)
{
        struct gpiohandle_request req;
        int ret;

        req.flags = flags;
        req.lines = 1;
        req.lineoffsets[0] = gpio;
        req.default_values[0] = 0;
        strcpy(req.consumer_label, label);

        ret = ioctl(fd, GPIO_GET_LINEHANDLE_IOCTL, &req);
        if (ret < 0) {
                printf("GPIO_GET_LINEHANDLE_IOCTL failed.\n");
                return -1;
        }

        *req_fd = req.fd;

        return 0;
}
```

### Read GPIO line

The value is read and printed from the line corresponding to the requested gpio number.

```cpp
int get_value(int req_fd, int gpio, int *value)
{
        struct gpiohandle_data data;
        int ret;

        memset(&data, 0, sizeof(data));

        ret = ioctl(req_fd, GPIOHANDLE_GET_LINE_VALUES_IOCTL, &data);
        if (ret < 0) {
                printf("GPIO_GET_LINE_VALUES_IOCTL failed.\n");
                return -1;
        }
        printf("line %d is %s\n", gpio, data.values[0] ? "high" : "low");

        *value = data.values[0];

        return 0;
}
```
### Write GPIO line

Prints the value of the line corresponding to the requested gpio number.

```cpp
int set_value(int req_fd, int gpio, int value)
{
        struct gpiohandle_data data;
        int ret;

        memset(&data, 0, sizeof(data));
        data.values[0] = value;

        ret = ioctl(req_fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &data);
        if (ret < 0) {
                printf("GPIO_SET_LINE_VALUES_IOCTL failed.\n");
                return -errno;
        }
 
 printf("line %d is %s\n", gpio, data.values[0] ? "high" : "low");

        return 0;
}
```

## Event

### Request GPIO Event

Set the line corresponding to the requested gpio number to the input mode and prepare to receive events. In the this example, the pin is set to the open-drain and respond evenly to both of edges(rising and falling) events.

```cpp
int request_event(int fd, int gpio, const char *label, int *req_fd)
{
        struct gpioevent_request req;
        int ret;

        req.lineoffset = gpio;
        strcpy(req.consumer_label, label);
        req.handleflags = GPIOHANDLE_REQUEST_OPEN_DRAIN;
        req.eventflags = GPIOEVENT_REQUEST_RISING_EDGE |
                GPIOEVENT_REQUEST_FALLING_EDGE;

        ret = ioctl(fd, GPIO_GET_LINEEVENT_IOCTL, &req);
        if (ret < 0) {
                printf("GPIO_GET_LINEEVENT_IOCTL failed.\n");
                return -1;
        }

        *req_fd = req.fd;

        return 0;
}
```

### Wait for GPIO Event

It wait for an event on the line corresponding to the requested gpio number.

```cpp
#define USE_POLL

int recv_event(int req_fd, int gpio)
{
        struct gpioevent_data event;
        struct pollfd pfd;
        ssize_t rd;
        int ret;

#ifdef USE_POLL
        pfd.fd = req_fd;
        pfd.events = POLLIN | POLLPRI;

        rd = poll(&pfd, 1, 1000);
        if (rd < 0) {
                printf("poll failed.\n");
                return -1;
		}
#endif

        ret = read(req_fd, &event, sizeof(event));
        if (ret < 0) {
                printf("read failed.\n");
                return -1;
        }

        printf( "GPIO EVENT @%" PRIu64 ": ", event.timestamp);

        if (event.id == GPIOEVENT_EVENT_RISING_EDGE)
                printf("RISING EDGE");
        else
                printf("FALLING EDGE");

        printf ("\n");

        return 0;
}
```

# GPIO Tools

For more approaches, see the `tools/gpio` in the kernel directory

## lsgpio

it shows the gpio line name and usage status after opening the gpio controller device with the name `gpiochip#`.

```bash
GPIO chip: gpiochip0, "pinctrl-bcm2835", 54 GPIO lines
 line 0: "[SDA0]" unused
 line 1: "[SCL0]" unused
 (...)
 line 16: "STATUS_LED_N" unused
 line 17: "GPIO_GEN0" unused
 line 18: "GPIO_GEN1" unused
 line 19: "NC" unused
 line 20: "NC" unused
 line 21: "CAM_GPIO" unused
 line 22: "GPIO_GEN3" unused
 line 23: "GPIO_GEN4" unused
 line 24: "GPIO_GEN5" unused
 line 25: "GPIO_GEN6" unused
 line 26: "NC" unused
 line 27: "GPIO_GEN2" unused
```

## gpio-hammer
```bash
$ gpio-hammer --help
gpio-hammer: invalid option -- '-'
Usage: gpio-hammer [options]...
Hammer GPIO lines, 0->1->0->1...
  -n <name>  Hammer GPIOs on a named device (must be stated)
  -o <n>     Offset[s] to hammer, at least one, several can be stated
 [-c <n>]    Do <n> loops (optional, infinite loop if not stated)
  -?         This helptext

Example:
gpio-hammer -n gpiochip0 -o 4

$ gpio-hammer -n gpiochip0 -o 5 -c 2
Hammer lines [5] on gpiochip0, initial states: [1]
[\] [5: 1]
```

## gpio-event-mon

```bash
$ gpio-event-mon --help
gpio-event-mon: invalid option -- '-'
Usage: gpio-event-mon [options]...
Listen to events on GPIO lines, 0->1 1->0
  -n <name>  Listen on GPIOs on a named device (must be stated)
  -o <n>     Offset to monitor
  -d         Set line as open drain
  -s         Set line as open source
  -r         Listen for rising edges
  -f         Listen for falling edges
 [-c <n>]    Do <n> loops (optional, infinite loop if not stated)
  -?         This helptext

Example:
gpio-event-mon -n gpiochip0 -o 4 -r -f
$ gpio-event-mon -n gpiochip0 -o 6 -r -c 2
Monitoring line 6 on gpiochip0
Initial line value: 1
GPIO EVENT 1527053280817909942: rising edge
GPIO EVENT 1527053282195388044: rising edge
```

# Reference

- [GPIO Subsystem -4- (new Interface)](http://jake.dothome.co.kr/gpio-4/)
