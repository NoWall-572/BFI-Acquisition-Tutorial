# BFI-Acquisition-Tutorial

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

波束赋形反馈是一种“按需”机制。我们需要制造压力来诱发它。

### 方案 A：使用 iperf3 (推荐 ⭐)
iperf3 数据量大，BFM 触发频繁。

1.  **服务端**（在局域网内的 PC 或 Kali 有线端）：
    ```bash
    iperf3 -s
    ```
2.  **客户端**（在 Wi-Fi 6 手机/STA 上）：
    *   手机安装 iperf3 App（如 "Magic iPerf" 或 Termux）。
    *   执行下行灌包命令：
    ```bash
    # -c [服务端IP] -R (反向模式，Server发给Client) -t 60 (持续60秒)
    iperf3 -c 192.168.1.X -R -t 60
    ```
    *   **注：** `-R` 参数非常重要，它产生下行流量，促使 AP 发送 NDPA，进而诱导 STA 回复 BFM。

### 方案 B：使用 Ping
*仅用于连通性测试，很难触发 BFM，因为包太小。*

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
