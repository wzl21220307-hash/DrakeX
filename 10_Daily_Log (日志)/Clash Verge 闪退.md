# 🛠️ BUG 排查记录：Clash Verge 闪退
📅 **时间**: 2026-04-12 15:11
🏷️ **标签**: #Troubleshooting #solved 

## 1. 错误现象 (Log)
> [!ERROR] 终端报错信息
> Gdk-Message: Error 71 (Protocol error) dispatching to Wayland display

## 2. 环境复核
- [ ] Ubuntu 22.04 LTS
- [ ] Wayland 显示协议（默认）
- [ ] Clash Verge 2.4.7
- [ ] Is Admin: false（非管理员权限运行）
## 3. 解决尝试
- 尝试 1: 切换到 X11 会话（Ubuntu on Xorg）→ 有效但不方便
- 尝试 2: 使用 sudo 启动 → 仍然闪退，Wayland 问题未解决
- 最终解决：
> [!SUCCESS]  更换为 nekoray（Qt 框架，Wayland 兼容性好）
```bash
删除 Clash Verge
sudo pkill -f clash
sudo apt remove --purge clash-verge -y
sudo rm -rf ~/.config/clash-verge
>
安装 nekoray
mkdir -p ~/Applications && cd ~/Applications
wget https://github.com/MatsuriDayo/nekoray/releases/download/3.26/nekoray-3.26-2023-12-09-linux64.zip 
unzip nekoray-3.26-2023-12-09-linux64.zip
sudo apt install -y libqt5core5a libqt5gui5 libqt5network5 libqt5widgets5 libqt5svg5 libqt5x11extras5
cd nekoray && ./nekoray
```
## 4. 根因复盘
- Tauri 框架依赖 WebKitGTK，在 Wayland 下有内存管理 Bug
- Linux 优先选用 Qt 框架的代理软件