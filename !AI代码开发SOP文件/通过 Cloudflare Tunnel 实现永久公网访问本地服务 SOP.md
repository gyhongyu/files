# **通过 Cloudflare Tunnel 实现永久公网访问本地服务 SOP**

## **一、 目标**

本 SOP 旨在指导用户如何使用 Cloudflare Tunnel，将运行在个人电脑或内网服务器上的本地服务（如 http://localhost:8000），安全、稳定地发布到公网，并通过自定义域名进行永久访问。

**核心优势**:

* **高安全性**: 无需暴露公网 IP，无需在路由器上打开任何端口。  
* **高稳定性**: 服务化运行，开机自启，断网自动重连。  
* **免费增值**: 自动获得 HTTPS 加密、CDN 加速及基础 DDoS 防护。

## **二、 先决条件**

在开始操作前，请确保您已具备以下条件：

1. **Cloudflare 账户**: 一个已注册的免费或付费 Cloudflare 账户。  
2. **个人域名**: 拥有一个您自己注册的域名（例如 your-domain.com）。  
3. **域名已托管**: 已将您的域名添加到 Cloudflare，并根据指引将域名的 Nameservers (NS) 记录修改为 Cloudflare 指定的服务器地址。  
4. **本地服务运行中**: 您的电脑上有一个正在运行的网络服务，并知道其访问地址（例如 http://localhost:8000）。

## **三、 操作步骤**

### **阶段一：安装并授权 Cloudflare 命令行工具 (cloudflared)**

此阶段在您的本地电脑上完成，只需操作一次。

1. **下载并安装 cloudflared**  
   * 访问 [Cloudflare 官方下载页面](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)，根据您的操作系统（Windows/macOS/Linux）下载对应的 cloudflared 程序并将其放置在易于访问的路径。  
2. **授权 cloudflared**  
   * 打开您电脑的命令行工具（Windows Terminal, PowerShell, macOS Terminal 等）。  
   * 运行以下命令：  
     cloudflared tunnel login

   * 此命令会自动打开一个浏览器窗口，要求您登录 Cloudflare 账户。选择您要使用的域名，点击 "Authorize"（授权）。  
   * 授权成功后，cloudflared 会在您的用户目录下生成一个 cert.pem 凭证文件，后续操作会自动使用该凭证。

### **阶段二：创建并配置永久隧道**

此阶段为您的服务创建一个永久性的隧道，并将其连接到您的域名。

1. **创建隧道**  
   * 在命令行中，为您的隧道指定一个易于记忆的名称（例如 my-app-tunnel），并运行以下命令：  
     cloudflared tunnel create my-app-tunnel

   * **关键输出**: 此命令会返回一个隧道的 **UUID**，并会在当前目录下创建一个与该 UUID 同名的 **JSON 凭证文件**（例如 xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json）。这个 JSON 文件是隧道的密钥，请妥善保管。  
2. **将域名路由到隧道**  
   * 为您的服务规划一个子域名（例如 service.your-domain.com），然后运行以下命令，将此域名指向您刚刚创建的隧道。**强烈建议使用子域名**，而不是直接使用根域名。  
     cloudflared tunnel route dns my-app-tunnel service.your-domain.com

   * 此命令会在您 Cloudflare 的 DNS 设置中自动创建一条 CNAME 记录，指向您的隧道。

### **阶段三：将隧道安装为系统服务（实现永久运行）**

这是实现“一劳永逸”的关键步骤。

1. **确保本地服务正在运行**  
   * 请确保您要发布的应用（例如 Docker 容器）已经启动。  
   * *示例 (Docker)*: docker-compose up \-d  
2. **将隧道安装为服务**  
   * 打开一个 **具有管理员权限** 的命令行窗口。  
     * *Windows*: 右键点击 "PowerShell" 或 “终端”，选择 "以管理员身份运行"。  
     * *macOS/Linux*: 在命令前加上 sudo。  
   * 运行以下命令，将隧道安装为系统服务。**注意**: 此命令需要 cloudflared 能够找到在**阶段二第1步**中创建的 JSON 凭证文件。请确保该文件与 cloudflared 在同一目录，或为其指定完整路径。  
     \# 将\<隧道名称\>替换为您自己的  
     cloudflared service install my-app-tunnel 

   * 命令执行成功后，隧道服务会立即在后台启动，并且会自动设置为开机自启。

## **四、 验证与管理**

1. **验证**  
   * 打开浏览器，访问您在**阶段二第2步**中设置的域名（例如 https://service.your-domain.com）。  
   * 如果能成功看到您本地的服务界面，则表示部署已全部完成。  
2. **服务管理 (备用)**  
   * 在需要时，您可以通过以下命令管理后台服务（同样需要管理员权限）：  
   * **停止服务**:  
     cloudflared service stop my-app-tunnel

   * **启动服务**:  
     cloudflared service start my-app-tunnel

   * **卸载服务** (将移除开机自启):  
     cloudflared service uninstall my-app-tunnel

**SOP 结束**