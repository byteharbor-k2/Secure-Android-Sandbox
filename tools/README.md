# 🛠️ 工具脚本使用说明

## 代理配置脚本

### setup_proxy.sh - 配置全局代理
通过ADB设置系统分身的全局HTTP代理，强制所有应用流量走电脑端代理。

**使用方法**:
```bash
cd /home/howie/Workspace/personal/Secure-Android-Sandbox/tools
./setup_proxy.sh
```

### clear_proxy.sh - 清除代理配置
恢复直连网络，清除代理设置。

**使用方法**:
```bash
./clear_proxy.sh
```

---

## ✅ 验证代理是否生效

### 方法1: 浏览器测试
在手机分身系统中打开浏览器，访问：
- `http://ip-api.com/json` - 查看出口IP（应显示代理服务器的IP）
- `https://www.google.com` - 测试是否能访问被墙网站

### 方法2: ADB验证
```bash
# 查看当前代理配置
adb shell settings get global http_proxy

# 测试HTTP请求（通过代理）
adb shell "curl -I http://www.google.com"
```

### 方法3: 查看Sparkle连接日志
在Sparkle/mihomo的控制面板（通常是 `http://127.0.0.1:9090/ui`）查看是否有来自手机的连接。

---

## ⚠️ 常见问题排查

### 问题1: 手机无法连接代理

**排查步骤**:
```bash
# 1. 确认电脑IP是否正确
ip -4 addr show | grep 192.168

# 2. 确认mihomo端口正在监听
ss -tlnp | grep 7890

# 3. 确认防火墙未阻止
sudo ufw status
# 如需开放端口:
sudo ufw allow 7890/tcp

# 4. 测试局域网连通性（在手机上执行）
ping 192.168.1.138
```

### 问题2: 某些应用不走代理

**原因**: 部分国产应用会忽略系统代理设置，直接建立TCP连接。

**解决方案**:
1. 使用VPN模式（需在手机上安装Clash/V2rayNG等客户端）
2. 使用透明代理（需要root权限或通过路由器配置）

### 问题3: WiFi代理配置后无法上网

**排查步骤**:
1. 确认电脑上的Sparkle正在运行
2. 确认"允许局域网连接"已开启
3. 尝试关闭代理后测试网络（排除WiFi本身问题）
4. 检查代理端口是否填写正确（7890）

### 问题4: 代理速度很慢

**可能原因**:
- WiFi信号弱
- 代理服务器节点拥堵
- mihomo TUN模式与手动代理冲突

**建议**:
- 在Sparkle中切换更快的节点
- 尝试使用WiFi 5GHz频段

---

## 📚 参考资料

- mihomo配置文件: `~/.config/clash/config.yaml`
- 端口说明:
  - `7890`: HTTP/HTTPS 代理端口
  - `9090`: Web控制面板端口（浏览器访问 `http://127.0.0.1:9090/ui`）

---

**更新日期**: 2026-02-12
