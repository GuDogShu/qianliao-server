# 牵聊 · 牵线服务器

给「牵聊」用的牵线服务器。它只做三件事：**登记地址、暂存密文、转发密文**。

消息在手机端就已经加密，服务器经手的全是密文，解不开，也看不到群成员和联系人关系。
它能看到的只有信封上的元数据：谁在什么时候找过谁、消息多大。

这个仓库**只发布编译好的安装包，不提供源码**。

---

## 安装

一台有公网 IP 的 Linux（Ubuntu 22.04 / Debian 12 最省事，1 核 512MB 够几十人用）。

```bash
curl -fsSL https://github.com/GuDogShu/qianliao-server/releases/latest/download/install.sh | sudo bash
```

装完会打印一行「让朋友在 App 里填这个地址」：

```
http://你的IP:8080
```

打开 App →「我」→「牵线服务器」→ 粘贴 → 添加。**聊天双方要填同一个地址。**

换端口（会同时换成 9000 / 9001 / 9002）：

```bash
curl -fsSL .../install.sh | sudo bash -s -- --port 9000
```

习惯先看脚本的，把 `| sudo bash` 换成 `-o install.sh`，看完再 `sudo bash install.sh`。

不想用脚本：下载 `qianliao-server.tar.gz`，解压后 `sudo bash deploy.sh`。
包里是单文件可执行，**目标机器不需要装 Node**。

---

## 三个端口

| 端口 | 用途 |
| --- | --- |
| 8080/tcp | 登记、查找、离线信箱 |
| 8081/tcp | 中继（转发密文） |
| 8082/udp | 打洞用的地址观测 |

云服务器还要在控制台「安全组」里放行这三个。漏了的表现是：服务器本机 `curl` 正常，手机连不上。

---

## 确认装的东西没被改过

三道校验，从外到内：

**1. 传输完整性**　`qianliao-server.tar.gz.sha256`：

```bash
sha256sum -c qianliao-server.tar.gz.sha256
```

**2. 来源真实性**　分离签名 + 发布公钥，指纹：

```
6A16 3013 A0CE 5742 863A DAA7 75E7 CE17 4CC1 5D82
```

```bash
gpg --import qianliao-release-key.asc
gpg --verify qianliao-server.tar.gz.asc qianliao-server.tar.gz
```

`install.sh` 里内置了这把公钥，验不过会直接停。

**3. 运行期自校验**　`integrity.json` 记录了可执行文件的 SHA256，服务每次启动先算自己的哈希，
对不上就拒绝启动（退出码 78）。随时手动查：

```bash
sudo -u qianliao /opt/qianliao-server/qianliao-server --check
```

边界说明：能改可执行文件的人通常也能改 `integrity.json`，所以这一层挡的是
「安装后被悄悄替换」「下载损坏」「镜像里被塞了东西」，挡不住已经拿到 root 的人。
真正兜底的是架构——服务端从头到尾只经手密文。

---

## 加固做了什么

`deploy.sh` 写出的 systemd 单元默认：

- 专用无登录用户运行，代码目录 `root:root` 只读，数据目录 `700`
- `ProtectSystem=strict` + `ReadOnlyPaths`，服务能写的只有数据目录
- `NoNewPrivileges` / `PrivateTmp` / `PrivateDevices` / `ProtectHome` / `ProtectKernel*`
- `CapabilityBoundingSet=` 清空能力集，`RestrictAddressFamilies` 只留 IP 与 Unix socket
- `UMask=0077`，落盘文件默认不对其他人开放

另外：**零第三方依赖**。服务端只用运行时自带的模块，安装过程中不从 npm 拉任何包，
不存在依赖投毒这条路径。

---

## 常用命令

```bash
sudo systemctl status qianliao     # 状态
sudo journalctl -u qianliao -f     # 日志
sudo systemctl restart qianliao    # 重启
```

- 数据：`/opt/qianliao-server/data/registry.json`（用户表、群目录、未取走的密文）
- 备份：复制这个文件即可
- 升级：重跑安装命令，数据不动
- 卸载：
  ```bash
  sudo systemctl disable --now qianliao
  sudo rm -f /etc/systemd/system/qianliao.service
  sudo rm -rf /opt/qianliao-server
  sudo userdel qianliao
  ```

---

## 客户端在哪

客户端是另一款应用，不在这个仓库。这里只有服务端安装包。

## 许可

安装包可自由用于自建服务；不提供源码，也不授权反编译后再分发。
