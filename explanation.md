
# uart_test.c Explanation

Library includes and The constants define the hardware path (/dev/ttyS0) and the standard baud rate (115200).
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <termios.h>
#include <errno.h>
#include <sys/select.h>

#define SERIAL_PORT "/dev/ttyS0" // Default UART in QEMU Virt
#define BAUD_RATE B115200
```

The program uses the `open()` system call to obtain description for UART device. 
The `O_NOCTTY` flag is used to disable controlling terminal

```c

int main() {
    int fd;
    struct termios tty;

    // 1. Open the device
    // O_RDWR: Read/Write, O_NOCTTY: Not a controlling terminal
    fd = open(SERIAL_PORT, O_RDWR | O_NOCTTY);
    if (fd < 0) {
        perror("Error opening UART");
        return EXIT_FAILURE;
    }
```

transfeering current config into local termios config
`tcgetattr() -> termios struct`
and adding default configuration

```c
    if (tcgetattr(fd, &tty) != 0) {
        perror("Error from tcgetattr");
        return EXIT_FAILURE;
    }

    cfsetospeed(&tty, BAUD_RATE);
    cfsetispeed(&tty, BAUD_RATE);

    tty.c_cflag = (tty.c_cflag & ~CSIZE) | CS8; // 8-bit chars
    tty.c_cflag |= (CLOCAL | CREAD);           // Ignore modem, enable reading
    tty.c_cflag &= ~(PARENB | PARODD);         // No parity
    tty.c_cflag &= ~CSTOPB;                    // 1 stop bit
    tty.c_cflag &= ~CRTSCTS;                   // No hardware flow control

    // Set to Raw Mode (disable canonical processing, echoing, etc.)
    tty.c_lflag &= ~(ICANON | ECHO | ECHOE | ISIG);
    tty.c_iflag &= ~(IXON | IXOFF | IXANY);
    tty.c_oflag &= ~OPOST;

    if (tcsetattr(fd, TCSANOW, &tty) != 0) {
        perror("Error from tcsetattr");
        return EXIT_FAILURE;
    }
```

Transferring data, The `write()` system call is used to push a string buffer to the UART
```c
    const char *msg = "RISC-V UART Test: Link Established\n";
    write(fd, msg, strlen(msg));
```
Non-blocking Receive using `select()`    
```c
    printf("Waiting for data (5s timeout)...\n");
    fd_set readfds;
    struct timeval timeout;
    
    FD_ZERO(&readfds);
    FD_SET(fd, &readfds);
    
    timeout.tv_sec = 5;
    timeout.tv_usec = 0;
```
If `select()` reports that data is available, the `read()` function pulls the bytes into a local buffer for display.

```c
    int ready = select(fd + 1, &readfds, NULL, NULL, &timeout);

    if (ready == -1) {
        perror("select() error");
    } else if (ready == 0) {
        printf("Timeout: No data received.\n");
    } else {
        char buf[256];
        int n = read(fd, buf, sizeof(buf) - 1);
        if (n > 0) {
            buf[n] = '\0';
            printf("Received: %s\n", buf);
        }
    }
```

`close()` is called to release the file descriptor and allow the OS to reclaim the UART resource for other processes.

```c
    close(fd);
    return EXIT_SUCCESS;
```
