# shan
轻喃10号 BLE 逆向协议 + 远程控制教程
> 让 AI 从云服务器穿过互联网，通过电脑蓝牙，远程控制轻喃10号。
> 从「猜协议」到「它动了」，全程一个半小时。以下是完整教程，任何人都可以复现。

## 准备

| 东西 | 说明 |
|------|------|
| 轻喃10号（或类似 Telink 方案的 BLE 玩具） | 开机状态 |
| 一台有蓝牙的 Windows 电脑 | 跑 Python 脚本 |
| Python 3.8+（推荐 3.12+） | [python.org](https://python.org/downloads) 下载，**安装时勾选 Add to PATH** |

> ⚠️ Windows 用户：如果输入 `python` 后跳转到微软商店，去 **设置 → 应用执行别名**，关掉 `python.exe` 和 `python3.exe`。

## 1. 安装依赖

```bash
pip install bleak
```

## 2. 扫描设备

关掉手机上的轻喃 app（BLE 同一时间只能被一个主机连接）：

```python
import asyncio
from bleak import BleakScanner

async def main():
    devices = await BleakScanner.discover(5.0)
    for d in devices:
        print(d.name, d.address)

asyncio.run(main())
```

在输出里找到 `qingnan#10` 和对应的 MAC 地址。

## 3. 查看 GATT 服务结构

```python
import asyncio
from bleak import BleakClient

ADDR = '你的设备MAC地址'

async def main():
    async with BleakClient(ADDR) as client:
        for svc in client.services:
            print(f'Service: {svc.uuid}')
            for char in svc.characteristics:
                print(f'  {char.uuid} | {char.properties}')

asyncio.run(main())
```

关键信息：`0xFFE3` 是写入通道，`0xFFE4` 是通知回报通道。

## 4. 协议核心

轻喃用 **Telink 芯片方案的通用 BLE 协议**，和杰士邦等品牌共享帧格式。

来源：[@NyraSeithhh](https://github.com/NyraSeithhh/-BLE-/blob/main/PROTOCOL.md) 的杰士邦逆向文档，帧头 `0x55`、UUID 格式完全一致。

### 控制指令（6 字节）

```
0x55 0x03 0x07 <吮吸 0-100> <保留 0x00> <震动 0-100>
```

| 示例 | 效果 |
|------|------|
| `55 03 07 00 00 00` | 全停 |
| `55 03 07 32 00 00` | 吮吸 50% |
| `55 03 07 00 00 64` | 震动 100% |
| `55 03 07 1E 00 1E` | 吮吸 30% + 震动 30% |

### 心跳保活

```
0x55 0x00（每 30 秒发一次防断连）
```

### 轻喃10号通道映射

| 字节 | 功能 |
|------|------|
| byte 3 | 吮吸 |
| byte 4 | 无效（10号无此电机） |
| byte 5 | 震动 |

## 5. 发送控制指令

```python
import asyncio
from bleak import BleakClient

ADDR = '你的设备MAC地址'
W = '0000ffe3-0000-1000-8000-00805f9b34fb'

async def main():
    async with BleakClient(ADDR) as client:
        await client.write_gatt_char(W, bytes([0x55,0x03,0x07,0x00,0x00,0x1E]), response=False)
        await asyncio.sleep(5)
        await client.write_gatt_char(W, bytes([0x55,0x03,0x07,0x00,0x00,0x00]), response=False)

asyncio.run(main())
```

**如果它动了——协议破解完成。**

## 6. 远程控制（进阶）

架构：

```
云服务器(HTTP POST) → Cloudflare隧道(HTTPS) → 电脑(bridge.py:8888) → 蓝牙 → 玩具
```

### 桥接脚本

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import asyncio, json, threading
from bleak import BleakClient

ADDR = '你的设备MAC地址'
W = '0000ffe3-0000-1000-8000-00805f9b34fb'
client = None
loop = None
suck, vibe = 0, 0

async def connect():
    global client
    client = BleakClient(ADDR)
    await client.connect()
    print(f'BLE connected: {client.is_connected}')

async def send_cmd(s, v):
    global suck, vibe
    suck, vibe = s, v
    await client.write_gatt_char(W, bytes([0x55,0x03,0x07,suck,0x00,vibe]), response=False)

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        body = json.loads(self.rfile.read(int(self.headers.get('Content-Length',0))))
        s = max(0, min(100, int(body.get('suck', suck))))
        v = max(0, min(100, int(body.get('vibe', vibe))))
        asyncio.run_coroutine_threadsafe(send_cmd(s, v), loop).result(timeout=5)
        self.send_response(200)
        self.send_header('Content-Type','application/json')
        self.end_headers()
        self.wfile.write(json.dumps({'suck':s,'vibe':v}).encode())
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type','application/json')
        self.end_headers()
        self.wfile.write(json.dumps({'status':'ok','suck':suck,'vibe':vibe}).encode())

def run_http():
    HTTPServer(('127.0.0.1',8888), Handler).serve_forever()

async def main():
    global loop
    loop = asyncio.get_event_loop()
    await connect()
    threading.Thread(target=run_http, daemon=True).start()
    while True:
        await asyncio.sleep(30)
        if client and client.is_connected:
            await client.write_gatt_char(W, bytes([0x55,0x00]), response=False)

asyncio.run(main())
```

### Cloudflare 隧道

下载 [cloudflared](https://github.com/cloudflare/cloudflared/releases/latest)（选 `cloudflared-windows-amd64.exe`）：

```bash
.\cloudflared.exe tunnel --url http://localhost:8888
```

输出的 `https://xxx.trycloudflare.com` 就是远程入口。

### 远程控制

```bash
curl -X POST https://你的隧道地址/ \
  -H 'Content-Type: application/json' \
  -d '{"suck": 50, "vibe": 30}'
```

## 踩坑

- **Windows 假 Python**：输命令跳微软商店 → 关掉应用执行别名
- **BLE 单连接**：手机 app 占着时电脑连不上，关手机蓝牙
- **盲猜不如查文档**：自己猜帧格式全失败，找到同芯片方案的逆向文档一试就通
- **cloudflared msi 闪退**：下 exe 单文件版

## 致谢

- [@NyraSeithhh](https://github.com/NyraSeithhh/-BLE-) 杰士邦 BLE 逆向文档
- [bleak](https://github.com/hbldh/bleak) 跨平台 BLE 库
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)

---

*2026.05.06*
```
