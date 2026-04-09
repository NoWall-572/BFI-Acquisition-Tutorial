# BFI采集教程

<div align="left">

[![紫金山实验室](https://img.shields.io/badge/紫金山实验室-PML-7030A0?style=for-the-badge)](https://www.pmlabs.com.cn/)
[![东南大学网络空间安全学院](https://img.shields.io/badge/东南大学-网络空间安全学院-005A3C?style=for-the-badge)](https://cyber.seu.edu.cn/)
[![东南大学移动通信国家重点实验室](https://img.shields.io/badge/东南大学-移动通信国家重点实验室-005A3C?style=for-the-badge)](https://ncrl.seu.edu.cn/)

</div>

<br>
本教程由紫金山实验室“五色石计划”特别支持。

---

## 〇、 前置准备 🛠️

在开始之前，必须确保硬件满足 **802.11ax** 的采集条件，否则无法捕获 HE 格式的帧。

1.  **嗅探网卡 (Sniffer)：**
    *   普通的 Wi-Fi 网卡（如 TL-WN722N 等 11n/ac 网卡）**无法捕获** 802.11ax 的数据包。
    *   必须使用**支持 802.11ax** 的网卡，并配合支持 Monitor Mode 的 Linux 驱动。
2.  **实验设备：**
    *   **AP (Beamformer):** 一台支持 Wi-Fi 6 (802.11ax) 的路由器（例如：TP-LINK BE7200）。
    *   **STA (Beamformee):** 一台支持 Wi-Fi 6 的手机（建议使用安卓，尽量避开苹果）或笔记本电脑。
    *   **控制端:** 安装了 Kali Linux 的电脑（用于运行嗅探工具和 iperf 服务端）。

---

## 一、 实验拓扑与连接 🌐

为了最大限度地诱发波束赋形（Beamforming），我们需要在 AP 和 STA 之间建立高吞吐量的**下行数据流** (Downlink Traffic)。因为只有当 AP 需要向 STA 发送大量数据时，AP 才有动力发起信道探测 (Sounding) 以优化下行链路。

### 连接步骤：

1.  **配置 AP：**
    *   登录路由器后台（TP-LINK 管理网址通常为 `tplogin.cn`）。
    *   **关闭“双频合一”**，并关闭 2.4GHz 频段。
    *   **固定信道 (Channel)：** 建议选择干扰较少的 5GHz 信道，例如 **36, 44, 149 或 153**。不要设为“自动”，推荐统一设置为 **153**。
    *   **固定频宽 (Bandwidth)：** 设置为 **80MHz**（实验最常用，便于分析）。
    *   确保开启 **"MU-MIMO"** 和 **"OFDMA"** 选项。
2.  **连接 STA：**
    *   将 Wi-Fi 6 手机/笔记本连接到该 AP 的 5GHz 频段。
    *   在管理界面确认设备已采用 WiFi 6 模式接入，并**记录 STA 的 MAC 地址**。
3.  **连接嗅探机 (Kali)：**
    *   将 USB 无线网卡插入电脑并连接到 Kali 虚拟机。
    *   确保 Kali 也能通过有线或其他方式连接到同一局域网（用于运行 iperf3 服务端）。

---

## 二、 流量诱导（诱发 BFM 产生）🚀

在 Wi-Fi 6 (802.11ax) 协议中，波束赋形反馈（Beamforming Feedback）是一种**“按需触发”**的底层机制。只有当路由器（AP）需要向目标设备（STA）发送大量数据，且面临复杂的信道环境时，AP 才有动力发起信道探测（Sounding）以优化下行链路。

因此，我们需要人为制造极限的**下行网络压力**来诱发 BFM 矩阵的产生。

### 方案 A：使用 iperf3 持续高频灌包 (强推 ⭐)

`iperf3` 能够产生极限吞吐量的数据流，是高频触发 BFM 的最佳工具。为了满足单用户（SU-MIMO）和多用户（MU-MIMO）的实验需求，请按照以下步骤操作：

#### 1. 启动服务端 (Server)
在通过有线连接到同一局域网的 PC 或 Kali 虚拟机上，打开终端并启动 iperf3 服务端，指定监听端口为 `5201`：
```bash
iperf3 -s -p 5201
```
*(注：如果需要测试双设备并发，建议开启两个独立的服务端实例，分别监听不同端口，例如再开一个终端运行 `iperf3 -s -p 5202`)*

#### 2. 启动客户端 (Client) 进行单设备通信 (SU-MIMO 触发)
在你的 Wi-Fi 6 接收端设备（如手机安装 Termux/Magic iPerf，或笔记本电脑终端）上，执行以下“无限量下行灌包”命令：
```bash
iperf3 -c <目的主机IP地址> -p 5201 -u -t 0 -b 0 -R
```

**🔑 核心参数硬核解析：**
*   `-c <目的主机IP地址>`：以客户端模式运行，并连接到刚才开启的服务端 IP。
*   `-p 5201`：指定通信端口为 5201。
*   `-u`：使用 **UDP** 协议。UDP 没有 TCP 的拥塞控制，能够以最大极限挤压带宽，是触发波束赋形的完美选择。
*   `-t 0`：设置传输时间为**无限（持续运行）**，直到手动 `Ctrl+C` 停止。
*   `-b 0`：取消带宽限制（要求 AP 以最大能力发送）。
*   `-R`：**Reverse（反向模式）**。这一步**极其重要**！它将默认的上行流量反转为**下行流量**（Server 发送给 Client），迫使 AP 主动进行波束赋形计算，从而诱导 STA 回复 BFM 包含矩阵。

#### 3. 进阶：双设备并发通信 (MU-MIMO 触发)
如果你需要采集复杂环境下的多用户 BFM 交叠数据，需要两台 Wi-Fi 6 设备同时向 AP 请求大流量。
1. **服务端**同时运行两个端口：`iperf3 -s -p 5201` 和 `iperf3 -s -p 5202`。
2. **设备 A (STA 1)** 连接端口 5201：
   ```bash
   iperf3 -c <目的主机IP地址> -p 5201 -u -t 0 -b 0 -R
   ```
3. **设备 B (STA 2)** 同时连接端口 5202：
   ```bash
   iperf3 -c <目的主机IP地址> -p 5202 -u -t 0 -b 0 -R
   ```
此时，路由器（AP）为了同时满足两台设备的高带宽下行需求，将高频触发 MU-MIMO 的信道探测，你将在嗅探端抓获海量的并发 BFM 数据。

---

### 方案 B：使用 Ping 进行连通性保活与基础测试

*⚠️ 注意：Ping 发送的是极小的 ICMP 报文，路由器通常会使用全向天线直接发送，极难触发复杂的波束赋形矩阵计算。此方案主要用于前期连通性验证或防止设备休眠。*

**操作步骤：**

1. **获取目的 IP：**
   * 登录路由器的管理后台（例如 TP-LINK 默认为 `tplogin.cn`）。
   * 进入“设备管理”或“DHCP 客户端列表”。
   * 找到你的实验目标设备（Wi-Fi 6 手机或笔记本），查看并记录下它当前被分配的局域网 IP 地址（例如 `192.168.1.105`）。

2. **执行持续 Ping 测试：**
   * 在局域网内的另一台 Windows 电脑上，按下 `Win + X` 打开 **Windows PowerShell**（或命令提示符 CMD）。
   * 输入以下命令向目的主机持续发送报文（`-t` 参数代表不间断发送，直到手动按下 `Ctrl+C` 停止）：
   ```powershell
   ping <目的主机IP> -t
   # 例如：ping 192.168.1.105 -t
   ```
3. **观察结果：** 只要终端持续返回 `来自 192.168.1.xxx 的回复: 字节=32 时间=...`，即代表无线链路连通性正常，可以随时切回 `iperf3` 开始正式的高频诱导抓包。

---

## 三、 Kali Linux 网卡配置与监听 💻

现在开始配置嗅探环境。

1.  **打开终端并检查网卡：**
    ```bash
    sudo ifconfig
    # 假设网卡名为 wlan0
    ```
2.  **设置监听模式 (Monitor Mode) 与信道：**
    *注意：建议使用 `iw` 指令，`iwconfig` 对 802.11ax 支持较差。*

    ```bash
    # 1. 关闭网卡
    sudo ip link set dev wlan0 down

    # 2. 设置为监听模式
    sudo iw wlan0 set type monitor

    # 3. 开启网卡
    sudo ip link set dev wlan0 up

    # 4. 设置信道和频宽 (关键步骤)
    # 语法：sudo iw dev <设备> set channel <信道> <频宽类型>
    sudo iw dev wlan0 set channel 153 80MHz
    ```
    *验证设置：输入 `iw dev` 查看当前信道和模式是否生效。*

---

## 四、 使用 Tshark 捕获 BFM 数据帧 🎣

在 iperf3 流量传输之前启动捕获。

**命令优化：** 为了防止抓包文件过大，我们通常只关注“动作帧”（Action Frames），因为 BFM 包含在其中。

```bash
# 捕获并存入文件
sudo tshark -i wlan0 -w /root/he_bfm_capture.pcap

# 或者使用实时行缓冲模式
sudo tshark -l -i wlan0 -w /root/he_bfm_capture.pcap
```

### 执行顺序：
1. 手机端准备好 `iperf3 ... -R` 打流。
2. Kali 运行 `tshark` 命令。
3. 手机开始打流。
4. 约 60 秒后，在 Kali 终端按 `Ctrl+C` 停止捕获。

---

## 五、 数据过滤与分析 (Wireshark) 🔍

这是最核心的一步，从海量数据中提取 Wi-Fi 6 的波束赋形报告。

1.  **打开文件：**
    ```bash
    wireshark /root/he_bfm_capture.pcap
    ```
2.  **应用显示过滤器 (Display Filter)：**
    *   **核心过滤器（针对 HE-MIMO BFM）：**
        `wlan.he.action.he_mimo_control`
    *   **组合过滤器（查看完整交互，含 NDPA 通告）：**
        `(wlan.fc.type_subtype == 21) || (wlan.he.action.he_mimo_control)`

3.  **识别正确的 BFM 帧：**
    在结果中寻找由 **STA (手机) 发送给 AP (路由器)** 的帧：
    *   **Protocol 列：** 显示为 `802.11` 或 `HE Action`。
    *   **Info 列：** 显示 `Action frame, Category: HE`。
    *   **展开 Frame 结构：**
        1. 展开 `IEEE 802.11 Wireless Management`
        2. 寻找 `HE Action code: HE Compressed Beamforming And CQI (0)`
        3. 展开 `HE MIMO Control` 字段。
        4. 展开 `Feedback Matrices`，即为所求的 **BFM 矩阵信息**。

4.  **解读 HE MIMO Control 数据：**
    *   **Nc Index / Nr Index:** 天线流数配置。
    *   **Channel Width:** 频宽（0=20, 1=40, 2=80MHz...）。
    *   **Codebook Size:** 反馈精度（如 4/2 bit 或 7/5 bit）。
    *   **Beamforming Report:** 紧随其后的十六进制数据，即压缩后的 $\phi$ 和 $\psi$ 角度矩阵。

---

## 六、 常见问题排查 (Troubleshooting) ❓

1.  **抓不到包 / 全是乱码 (FCS Error)：**
    *   **距离问题：** 嗅探器的位置非常关键！AP 信号强容易抓，但 **BFM 是由 STA 发出的**。如果网卡离手机太远，将收不到反馈帧。
    *   **建议：** 将嗅探网卡放置在**靠近手机（1米以内）**的地方。

2.  **看不到 HE Action / BFM 内容：**
    *   **加密问题：** 在 WPA2/WPA3 网络中，部分 Action Frame 是加密的（Protected Action Frame）。
    *   **解决方法：**
        *   **方法一（推荐）：** 路由器设置 **OWE (Enhanced Open)** 或完全 **Open (无密码)** 进行实验。
        *   **方法二：** 如果必须用 WPA，需在 Wireshark 协议设置中输入 Wi-Fi 密码，并完整捕获手机连接时的 **4-way handshake**。

3.  **tshark 报错 "Operation not supported"：**
    *   通常意味着驱动不支持 Monitor 模式或不支持 80MHz 频宽。请确认网卡芯片型号。

---

## 总结：BFM 采集过滤速查表 📝

| 步骤 | 关键命令/动作 |
| :--- | :--- |
| **设置网卡** | `sudo iw dev wlan0 set channel 153 80MHz` |
| **触发流量** | `iperf3 -c <Server_IP> -R` (下行流是关键) |
| **VHT 过滤** | `wlan.vht.mimo_control` (针对 Wi-Fi 5) |
| **HE 过滤** | `wlan.he.action.he_mimo_control` (针对 Wi-Fi 6) |
| **完整交互** | `wlan.fc.type_subtype == 21` |
