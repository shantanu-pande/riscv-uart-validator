
# RISC-V ACT Framework Enablement and M-Mode Firmware Validation on Hardware Board

Aim: Develop a C program that initializes and configures a UART interface (e.g., `/dev/ttyS0` or `/dev/ttyUSB0`) on Linux using the `termios` API.

Objective: Validate the hardware enablement of a UART interface on a RISC-V 64-bit architecture.

Why this approch: I do not have RISCV board in hand. I use QEMU because it allows developers without hardware. Once the software is proven perfectly in QEMU, then it is flashed onto the physical hardware for the final real-world validation.


## Environment Setup and installation

Clone the Repository

```Bash
git clone https://github.com/shantanu-pande/riscv-uart-validator
cd riscv-uart-validator
```

Step 1: Install the RISC-V Toolchain
```Bash
sudo apt update
sudo apt install build-essential libncurses5-dev rsync git wget cpio python3 unzip bc
sudo apt install gcc-riscv64-linux-gnu qemu-system-misc
``` 

Step 2: Cross-Compile the Program and test for errors. 
```bash
riscv64-linux-gnu-gcc -o uart_test uart_test.c -static
```

Step 3: Buildroot setup and compilation:

Buildroot require $PATH environment variable set.
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

## Image build

Clone buildroot
```bash
git clone https://github.com/buildroot/buildroot.git
cd buildroot

make qemu_riscv64_virt_defconfig
```

Create the overlay directory structure and include folder in build.
```bash
mkdir -p board/qemu/riscv64-virt/rootfs_overlay/usr/bin/

riscv64-linux-gnu-gcc -static ../uart_test.c -o board/qemu/riscv64-virt/rootfs_overlay/usr/bin/uart_test

echo 'BR2_ROOTFS_OVERLAY="board/qemu/riscv64-virt/rootfs_overlay"' >> .config
```

Image build:
```bash
make
```


Boot into qemu
```bash 
cd output/images
./start-qemu.sh
```

login as `root`

Execute code with 
```bash
/usr/bin/uart_test
```

## Transfering and executing updated firmware

```bash
cd ..
riscv64-linux-gnu-gcc -static uart_test.c -o uart_test
```

starting python server

```bash
python3 -m http.server 8000
```

Find Host ip (QEMU's default network maps the host machine to IP 10.0.2.2).

```bash
cd /tmp
wget http://10.0.2.2:8000/uart_test
chmod +x uart_test
./uart_test
```
## License

[MIT](https://choosealicense.com/licenses/mit/)


## Contributing

Contributions are always welcome!


## Authors

- [@shantanu-pande](https://github.com/shantanu-pande)

