-----------------------------------------------------------------

## nftables NAT 规则生成工具

用途：便捷地设置 NAT 转发

> 适用于支持 nftables 的 Linux 发行版本，如 Debian 12+

## 优势

1. 实现动态 NAT：自动探测配置文件和目标域名 IP 的变化，除变更配置外无需任何手工介入
2. 支持 IP 和域名
3. 支持单独转发 TCP 或 UDP
4. 支持转发到本机其他端口（NAT 重定向）【2023.1.17更新】
5. 以配置文件保存转发规则，可备份或迁移到其他机器
6. 自动探测本机 IP
7. 支持自定义本机 IP【2023.1.17更新】
8. 开机自启动
9. 支持端口段
10. 轻量，只依赖 Rust 标准库

## 准备工作

1. 关闭 firewalld
2. 关闭 selinux
3. 开启内核端口转发
4. 安装 nftables（一般情况下系统默认安装了 nftables）

以下一键完成：

```shell
# 关闭firewalld
service firewalld stop
systemctl disable firewalld
# 关闭selinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config  
# 修改内存参数，开启端口转发
echo 1 > /proc/sys/net/ipv4/ip_forward
sed -i '/^net.ipv4.ip_forward=0/'d /etc/sysctl.conf
sed -n '/^net.ipv4.ip_forward=1/'p /etc/sysctl.conf | grep -q "net.ipv4.ip_forward=1"
if [ $? -ne 0 ]; then
    echo -e "net.ipv4.ip_forward=1" >> /etc/sysctl.conf && sysctl -p
fi
# 确保nftables已安装
yum install -y  nftables
```

**Debian 系说明**

如果 nftables 没有安装，请使用 `apt install nftables` 安装，当然 iptables 无需卸载

## 使用说明

```shell
# 必须是 root 用户
# sudo su

systemctl stop nat
# 下载可执行文件并授予权限
chmod +x /usr/local/bin/nat

# 创建 systemd 服务
cat > /lib/systemd/system/nat.service <<EOF
[Unit]
Description=dnat-service
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/opt/nat
EnvironmentFile=/opt/nat/env
ExecStart=/usr/local/bin/nat /etc/nat.conf
LimitNOFILE=100000
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

# 设置开机启动，并启动该服务
systemctl daemon-reload
systemctl enable nat

mkdir /opt/nat
touch /opt/nat/env
# echo "nat_local_ip=10.10.10.10" > /opt/nat/env #自定义本机ip，用于多网卡的机器

# 生成配置文件，配置文件可按需求修改（请看下文）
cat > /etc/nat.conf <<EOF
SINGLE,49999,59999,baidu.com
RANGE,50000,50010,baidu.com
EOF

systemctl start nat
```

**自定义转发规则**

`/etc/nat.conf` 如下：

```
SINGLE,49999,59999,baidu.com
RANGE,50000,50010,baidu.com
```

- 每行代表一个规则，行内以英文逗号分隔为 4 段内容
- SINGLE：单端口转发：本机 49999 端口转发到 baidu.com:59999
- RANGE：范围端口转发：本机 50000-50010 端口段转发到 baidu.com:50000-50010
- 请确保配置文件符合格式要求，否则程序可能会出现不可预期的错误，包括但不限于您的服务器炸掉（认真

高级用法：

1. **转发到本地**：行尾域名处填写 localhost 即可，例如`SINGLE,2222,22,localhost`，表示本机的 2222 端口重定向到本机的 22 端口。
2. **仅转发 TCP/UDP 流量**：行尾增加 tcp/udp 即可，例如`SINGLE,10000,443,baidu.com,tcp`表示仅转发 TCP 流量，`SINGLE,10000,443,baidu.com,udp`仅转发 UDP 流量

如需修改转发规则，请 `vim /etc/nat.conf` 以设定您想要的转发规则。修改完毕后，无需重新启动服务器或服务，程序将会自动在最多一分钟内更新 NAT 转发规则（PS：受 DNS 缓存影响，可能会超过一分钟）

## 一些需要注意的东西

1. 本工具在 Debian 12 上有效，其它发行版未作测试
2. 与前作 [arloor/iptablesUtils](https://github.com/arloor/iptablesUtils) 不兼容，在两个工具之间切换时，请重装系统以确保系统纯净！

## 如何停止以及卸载

```shell
## 停止服务
service nat stop
## 清空 NAT 规则
nft add table ip anat
nft delete table ip anat
## 删除开机启动条目
systemctl disable nat
```

## 编译方法

```shell
cargo build --release
```

## 致谢

1. [解决会清空防火墙的问题](https://github.com/arloor/nftables-nat-rust/pull/6)
2. [Ubuntu18.04 适配](https://github.com/arloor/nftables-nat-rust/issues/1)

## 常问问题

### 用于多网卡的机器时，如何指定用于转发的本机 IP

可以执行以下脚本来自定义本机 IP，该示例是将本机 IP 定义为`10.10.10.10`

```shell
echo "nat_local_ip=10.10.10.10" > /opt/nat/env
```

### 如何查看最终的nftables规则

```shell
nft list ruleset
```

### 查看日志

执行

```shell
cat /opt/nat/log/nat.log
```

或执行

```shell
journalctl -exu nat
```
