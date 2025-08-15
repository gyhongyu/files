# **Windows Server Web 应用部署标准作业程序 (SOP)**

### **1.0 目的**

本文档旨在规范化在 Windows Server 环境下，将基于 Node.js 的全栈 Web 应用部署为稳定、开机自启的后台服务之标准流程。通过使用 PM2 进程管理器，确保应用的高可用性和易管理性。

### **2.0 适用范围**

本 SOP 适用于：

* **服务器环境**: Windows Server 2016 或更高版本。  
* **应用架构**: 基于 Node.js (Express) 的后端，搭配需要构建（Build）的前端框架（如 React, Vue, Angular）。  
* **部署目标**: 将应用注册为系统服务，实现服务器重启后应用自动运行，并提供便捷的管理脚本。

### **3.0 部署前准备**

在执行部署流程前，请确保以下条件已满足：

| 序号 | 准备事项 | 检查方法与标准 |
| :---- | :---- | :---- |
| 3.1 | **Node.js 环境** | 打开 PowerShell 或 CMD，输入 node \-v 和 npm \-v。应能正确显示版本号。若无，请前往 [Node.js 官网](https://nodejs.org/) 下载并安装 LTS 版本。 |
| 3.2 | **应用代码** | 完整的项目代码已上传至服务器的指定目录（例如 C:\\my-webapp）。 |
| 3.3 | **项目依赖安装** | 在项目根目录下，已成功运行 npm install 命令，node\_modules 文件夹已存在。 |
| 3.4 | **前端项目构建** | 在项目根目录下，已成功运行 npm run build 命令。一个名为 build 或 dist 的文件夹已生成，其中包含 index.html 等静态文件。 |
| 3.5 | **后端代码修改** | 已按照要求修改 server.js 文件，使其能够：\<br\>1. 托管前端静态文件 (build 目录)。\<br\>2. 监听 0.0.0.0 地址以接受公网访问。 |
| 3.6 | **防火墙端口** | 确认应用计划使用的端口号（例如 3001），并准备在 Windows 防火墙中为其添加入站规则。 |

### **4.0 作业流程**

#### **4.1 阶段一：安装并注册为系统服务（一次性操作）**

此阶段将应用永久性地注册为开机自启的 Windows 服务。**此流程仅需在首次部署时执行一次。**

操作人员: 服务器管理员  
所需权限: 管理员 (Administrator)  
步骤 1：创建安装脚本  
在您的项目根目录（C:\\my-webapp）下，创建名为 install\_service.bat 的文件，并将以下代码粘贴进去。  
@echo off  
TITLE WebApp Service Installer

:: 检查是否以管理员身份运行  
net session \>nul 2\>&1  
if %errorLevel% \== 0 (  
    echo Administrator permissions confirmed. Proceeding with installation...  
) else (  
    echo \=================================================================  
    echo ERROR: This script requires Administrator privileges.  
    echo Please right-click on the file and select "Run as administrator".  
    echo \=================================================================  
    pause  
    exit  
)

echo.  
echo \[Step 1/5\] Installing PM2 globally...  
call npm install pm2 \-g

echo.  
echo \[Step 2/5\] Installing PM2 Windows Startup tool...  
call npm install pm2-windows-startup \-g

echo.  
echo \[Step 3/5\] Starting your application with PM2...  
REM 切换到此脚本所在的目录  
cd /d %\~dp0  
call pm2 start server.js \--name my-webapp

echo.  
echo \[Step 4/5\] Saving the application list for auto-reboot...  
call pm2 save

echo.  
echo \[Step 5/5\] Registering PM2 as a startup service...  
call pm2-startup install

echo.  
echo \=================================================================  
echo SUCCESS\!  
echo Your application has been registered as a Windows service.  
echo It will now start automatically every time the server boots up.  
echo You can now use 'control\_app.bat' to manage it.  
echo \=================================================================  
echo.  
pause

**步骤 2：执行安装脚本**

1. 找到 install\_service.bat 文件。  
2. **右键点击该文件，选择 “以管理员身份运行”**。  
3. 脚本会自动完成所有工具的安装和服务的注册。请耐心等待其执行完毕，直到看到 "SUCCESS\!" 消息。

**步骤 3：配置防火墙**

1. 打开 "高级安全 Windows Defender 防火墙"。  
2. 添加入站规则，协议为 TCP，特定本地端口为您在 server.js 中设置的端口（例如 3001），操作为“允许连接”。

#### **4.2 阶段二：日常应用管理**

服务安装成功后，日常的启动、停止、重启和日志查看等操作，均通过管理脚本完成。

操作人员: 开发/运维人员  
所需权限: 普通用户  
步骤 1：创建管理脚本  
在您的项目根目录（C:\\my-webapp）下，创建名为 control\_app.bat 的文件，并将以下代码粘贴进去。  
@echo off  
TITLE WebApp Control Panel  
:menu  
cls  
echo \=================================  
echo      WebApp Control Panel  
echo \=================================  
echo.  
echo  1\. Restart Application  
echo  2\. Stop Application  
echo  3\. Start Application  
echo  4\. Show Application Status  
echo  5\. Show Live Logs  
echo  0\. Exit  
echo.

set /p choice="Enter your choice: "

if not '%choice%'=='' set choice=%choice:\~0,1%  
if '%choice%'=='1' goto restart  
if '%choice%'=='2' goto stop  
if '%choice%'=='3' goto start  
if '%choice%'=='4' goto status  
if '%choice%'=='5' goto logs  
if '%choice%'=='0' goto exit

echo Invalid choice, please try again.  
pause  
goto menu

:restart  
echo Restarting application...  
call pm2 restart my-webapp  
pause  
goto menu

:stop  
echo Stopping application...  
call pm2 stop my-webapp  
pause  
goto menu

:start  
echo Starting application...  
call pm2 start my-webapp  
pause  
goto menu

:status  
echo Showing application status...  
call pm2 list  
pause  
goto menu

:logs  
echo Showing live logs... (Press Ctrl+C to stop viewing)  
call pm2 logs my-webapp  
pause  
goto menu

:exit  
exit

**步骤 2：使用管理脚本**

* **日常操作**: 直接双击 control\_app.bat 文件即可打开管理菜单。  
* **代码更新后**: 当您更新了服务器上的代码后，只需运行此脚本并选择 1\. Restart Application 即可使新代码生效。  
* **故障排查**: 当应用无法访问时，运行此脚本并选择 5\. Show Live Logs 是排查问题的首选方式。

### **5.0 部署后验证**

1. **服务状态检查**: 运行 control\_app.bat 并选择 4\. Show Application Status。应能看到 my-webapp 状态为 online，status 为 online。  
2. **公网访问**: 在外部电脑的浏览器中，访问 http://\<您的服务器公网IP\>:\<端口号\> (例如 http://123.45.67.89:3001)。应能正常显示您的应用界面。  
3. **重启测试 (可选)**: 重启 Windows Server，待服务器启动完成后，再次检查服务状态和公网访问，以确认开机自启功能正常。