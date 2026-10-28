# alipay-bill-dist

**基于支付宝官方接口的服务器端「免签」对账方案** —— 不需要 Xposed、不需要通知监听，
服务器自己就能拉账单、核对到账。


**技术支持：superhide.tang@gmail.com**

---

## 这是什么

把支付宝「交易记录」页的账单，用**服务器本机的登录态**抓下来，缓存成结构化数据，
供你的订单系统核对「钱到底到没到」。

- **纯服务器端**：不需要手机常驻、不装 App、不挂 Xposed
- **抓的是支付宝官方页面接口**，不是逆向私有协议
- **多用户隔离**：每个账号一套独立会话与缓存，互不干扰
- **不落 Cookie 明文**：登录态由库加密托管，对外只暴露「有没有登录」

## 能力一览

| 能力 | 说明 |
| --- | --- |
| 抓账单 | 拉取支付宝「交易记录」首页账单（约最近一个月 20 条） |
| Cookie 保活 | 定期访问一次，靠服务端下发的 Set-Cookie 滚动续期 |
| 扫码登录 | 在**服务器本机**开无头浏览器出二维码，手机支付宝扫一下即完成登录 |
| 账单缓存 | 去重 + 只留最近 30 天，别的业务查缓存即可，不必反复打支付宝 |
| 邮件通知 | Cookie 失效 / 恢复时发邮件（多 SMTP 账号自动切换） |
| 页面改版容错 | 解析失败自动存快照 + 诊断信息，下次改版直接看快照定位 |

## 下载

到本仓库的 **Releases** 下载 `v1.0.0`：

| 平台 | 文件 |
| --- | --- |
| Linux x86_64 | `alipay_bill-1.0.0-cp39-abi3-manylinux2014_x86_64.whl` |
| Linux x86_64（glibc≥2.28） | `alipay_bill-1.0.0-cp39-abi3-manylinux_2_28_x86_64.whl` |
| macOS Apple Silicon | `alipay_bill-1.0.0-cp39-abi3-macosx_11_0_arm64.whl` |
| macOS Intel | `alipay_bill-1.0.0-cp39-abi3-macosx_10_9_x86_64.whl` |
| Windows x64 | `alipay_bill-1.0.0-cp39-abi3-win_amd64.whl` |
| Linux 完整体验包（含演示页面 + 说明） | `alipay-bill-1.0.0-manylinux2014_x86_64.tar.gz` |

> Python **3.9+**（abi3：同一个包通吃 3.9 ~ 3.13，同平台不必为每个版本各下各的）

## 安装

```bash
pip install alipay_bill-1.0.0-cp39-abi3-<平台>.whl

# 扫码登录需要额外装浏览器内核（约 800MB，一次性成本）
python3 -m pip install playwright
python3 -m playwright install chromium
# 国内网络慢时加镜像：
# PLAYWRIGHT_DOWNLOAD_HOST=https://npmmirror.com/mirrors/playwright \
#   python3 -m playwright install chromium
```

## 快速上手

**命令行**

```bash
export ALIPAY_BILL_LICENSE_CODE=ALB-XXXX-XXXX-XXXX
# 发行包已内置默认授权服务地址，通常无需设置；私有化部署时才需要：
# export ALIPAY_BILL_LICENSE_SERVER=https://你的授权服务

# 下面用模块名调用（安装后也有 alipay-bill 命令，二者等价）
python3 -m alipay_bill_sdk activate          # 首次激活（绑定本机）
python3 -m alipay_bill_sdk qr --out qr.png    # 扫码登录（手机扫 qr.png 确认）
python3 -m alipay_bill_sdk status             # 看状态
python3 -m alipay_bill_sdk refresh            # 真打一次支付宝，把账单并入缓存
python3 -m alipay_bill_sdk bills --csv bills.csv   # 导出账单
python3 -m alipay_bill_sdk serve              # 常驻：按间隔跑保活
```

**演示页面**（完整体验包里自带，多用户扫码 + 看最近账单）

```bash
tar xzf alipay-bill-1.0.0-manylinux2014_x86_64.tar.gz
cd alipay-bill-1.0.0-manylinux2014_x86_64/demo
./start.sh          # 首次会自动安装同级 wheel
# 浏览器打开 http://127.0.0.1:8080
```

**作为库嵌进你自己的系统**：替换存储（`BillStore`）与通知（`Notifier`）两个抽象即可，
接入示例见发行包内 `examples/`。

## 授权

一张授权码绑定一台机器，可限制处理的**账号数量**（按账号数分档授权，各档时长与价格
分开计算）。到期或被吊销后，客户端在离线宽限期（默认 2 小时）后停止工作。

## 为什么不走「Xposed / 通知监听」那条路

市面上常见的支付宝免签，是基于 Xposed 或通知监听：收到支付宝通知推送后，再由手机
App 转发到服务器做金额比对（作者另一款 Xposed 插件也支持这个玩法，可在此体验：
<https://github.com/Xposed-Modules-Repo/io.github.superhide.viptry>）。

但这条路**依赖手机网络环境与 App 稳定性**：手机息屏、断网、App 被杀、系统清理后台，
都会导致漏单；而且账单数据要经手机中转。

本项目把登录动作放在**服务器本机**完成，账单直接从服务器端拉取 —— 不需要手机常驻，
稳定性与数据链路都短得多。

## 个人体验

<https://42.194.133.137:8787/account.html> —— 选择「开发者模式」，用邮箱验证码登录即可。

> **浏览器会提示「证书不受信任」**：授权服务用的是 IP 直连的自签名证书（不是公网 CA 签的），
> 浏览器无法自动信任。请选择「继续访问 / 高级 → 前往」—— 客户端侧已做**证书固定**
> （只认这一张证书），不会因此降低中间人防护。换成域名 + 公有 CA 签发后此提示即消失。

## 合规说明

本项目调用的是支付宝官方页面接口，**不存在隐私泄漏**；但请**仅合规使用自己的
支付宝账号**，不要冒用他人账号。使用前请自行确认符合支付宝的服务条款与相关法律法规。

## 许可

专有软件，详见发行包内 `LICENSE`。使用、授权与商业分发请联系著作权人。

## 效果预览

<img width="898" height="941" alt="授权区域" src="https://github.com/user-attachments/assets/1b702a46-1681-46e8-9b77-57ae0101508c" />
<img width="759" height="784" alt="到账检测" src="https://github.com/user-attachments/assets/c0ea6a3a-fc15-4916-9103-dc351c3d2b14" />
<img width="822" height="798" alt="扫码登陆2" src="https://github.com/user-attachments/assets/4a1e9814-5b0e-4a96-ba73-3d8eb5e8056c" />
<img width="1082" height="834" alt="账单列表" src="https://github.com/user-attachments/assets/9234d392-4e8e-4957-a5f1-67035b035311" />

<img width="425" height="546" alt="手机端提示" src="https://github.com/user-attachments/assets/9b94ffd1-2921-4b58-a28c-7800f71bd82c" />
<img width="520" height="923" alt="手机体验区1" src="https://github.com/user-attachments/assets/666fc038-c6f6-47f2-a9cb-c5a030be9a0c" />
<img width="534" height="910" alt="手机体验区2" src="https://github.com/user-attachments/assets/c050b15e-df3c-4e56-ac81-b749e4a3cf2b" />
<img width="503" height="919" alt="手机体验区3" src="https://github.com/user-attachments/assets/f55206d5-baec-4d55-9060-1da38f4d2e0e" />


