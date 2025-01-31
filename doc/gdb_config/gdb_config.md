# OpenBMC 进行远程 GDB 调试

在 OpenBMC 开发过程中，需要使用 GDB 进行远程调试。由于初期可能没有开发板，本文首先在 QEMU 环境下进行配置说明，真实的开发板还没有进行测试过。以这个phosphor-network-manager为例。

## **1. 在 QEMU 环境下进行初步测试**

如果你在 QEMU 环境下进行调试，可以使用以下命令启动 QEMU，确保 BMC 设备的网络环境正确，并添加 GDB 需要的端口映射：

```bash
qemu-system-arm -M ast2600-evb \
    -nographic \
    -drive file=/home/lijilin/openbmc/build/evb-ast2600/tmp/deploy/images/evb-ast2600/obmc-phosphor-image-evb-ast2600.static.mtd,format=raw,if=mtd \
    -net nic \
    -net user,hostfwd=tcp::2222-:22,hostfwd=tcp::8443-:443,hostfwd=tcp::1234-:1234
```

### **QEMU 启动参数详解**

- `-M ast2600-evb`：指定 QEMU 运行 AST2600 评估板的模拟模式。

- `-nographic`：禁用图形界面，仅使用串口终端。

- `-drive file=...,format=raw,if=mtd`：加载 OpenBMC 镜像，模拟 SPI 闪存存储。

- `-net nic`：创建一个默认的网络接口。

- ```
  -net user,hostfwd=...：
  ```

  - `hostfwd=tcp::2222-:22`：将 QEMU 的 SSH 端口 22 映射到宿主机的 2222 端口。
  - `hostfwd=tcp::8443-:443`：将 HTTPS 端口 443 映射到宿主机的 8443 端口。
  - `hostfwd=tcp::1234-:1234`：将 GDB 端口 1234 映射到宿主机，以便远程调试。

### **确保 QEMU 网络环境与开发环境的连通性**

为了确保 QEMU 内部环境能够正确通信开发服务器，需要进行以下检查：

1. 检查 QEMU 的网络连接

   ```bash
   ping 10.0.2.15  # QEMU 默认分配的 IP
   ```

   如果无法 ping通，可能需要调整 QEMU 的 -net 参数或使用 TAP 网络桥接模式。

2. 确认端口映射生效

   ```bash
   nc -zv localhost 1234  # 测试 GDB 端口
   ```

   如果端口未开放，需要检查 QEMU 启动参数或防火墙配置。

3. 检查开发服务器防火墙设置

   ```bash
   sudo iptables -L
   ```

   确保没有规则阻止 1234端口的外部连接。

## **2. 在 BMC 设备上启动 GDBserver**

在 BMC 终端中运行以下命令启动 `gdbserver`：

```bash
root@evb-ast2600:/usr/bin# gdbserver :1234 phosphor-network-manager
```

如果要调试一个正在运行的进程，可以使用：

```bash
gdbserver :1234 --attach <PID>
```

此时，`gdbserver` 会监听 `1234` 端口等待调试器连接。

## **3. 在开发服务器上安装 GDB 并连接到 GDBserver**

如果开发服务器上没有 `gdb`，可以使用以下方式安装：

### **（1）安装 GDB（推荐使用 `gdb-multiarch`）**

对于 Ubuntu/Debian：

```bash
sudo apt update
sudo apt install gdb-multiarch -y
```

### **（2）使用 GDB 连接到 BMC 设备**

在开发服务器上，运行：

```bash
gdb-multiarch /path/to/phosphor-network-manager
```

然后在 GDB 命令行中输入：

```gdb
target remote <BMC_IP>:1234  # 真实开发板使用 BMC 的 IP 地址
```

如果连接成功，会看到类似：

```
Remote debugging using <BMC_IP>:1234
```

## **4. 解决调试符号丢失的问题**

### **（1）检查调试符号**

如果 GDB 提示 `No debugging symbols found`，说明 `phosphor-network-manager` 可能缺少调试信息。

可以检查目标文件是否包含符号：

```bash
file /path/to/phosphor-network-manager
```

如果输出不包含 `with debug_info`，说明符号被剥离，需要重新编译。

### **（2）重新编译并添加调试信息**

```bash
bitbake phosphor-network-manager -c clean
bitbake phosphor-network-manager -c compile -f -g
```

然后，确保调试信息存在：

```bash
find ~/openbmc/build/evb-ast2600/tmp/work/ -name "phosphor-network-manager"
```

## **5. 进行远程调试**

成功加载调试符号后，可以使用 GDB 进行调试，例如：

```gdb
b main      # 在 main() 入口点设置断点
c          # 继续运行
info threads # 查看线程信息
bt         # 打印调用栈
step       # 单步执行
next       # 逐行执行
continue   # 继续运行
```

## **6. 结束调试**

当调试完成后，可以在 BMC 设备上终止 `gdbserver` 进程，或者在 GDB 中输入：

```gdb
detach
quit
```

这样，BMC 设备上的 `phosphor-network-manager` 将继续运行，而 GDB 断开调试。

## **7. 结论**

本文介绍了如何在真实开发板上运行 OpenBMC 并使用远程 GDB 调试目标程序的完整流程，包括：

- 先在 QEMU 进行测试（可选）
- 在 BMC 设备上启动 `gdbserver`
- 在开发服务器上使用 `gdb-multiarch` 进行远程调试
- 解决调试符号丢失的问题
- 确保 QEMU 网络与开发服务器端口的连通性

通过这些步骤，你可以高效地在 OpenBMC 开发环境中进行远程 GDB 调试，提高调试效率！🚀