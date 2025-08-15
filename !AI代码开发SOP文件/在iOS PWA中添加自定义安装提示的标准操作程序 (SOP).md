# **在iOS PWA中添加自定义安装提示的标准操作程序 (SOP)**

## **一、 目标**

本SOP旨在指导您如何在您的渐进式网络应用 (PWA) 中集成一个自定义提示，以引导iOS用户将您的应用添加到其主屏幕，从而绕过iOS不提供原生PWA安装提示的限制。

## **二、 前置条件**

在开始之前，请确保您已具备以下条件：

1. **一个已部署的PWA应用**：您的PWA应用已通过HTTPS部署，并包含有效的manifest.json文件和Service Worker，已在Android和Windows Chrome上成功安装。  
2. **可访问您的PWA的HTML文件**：您需要能够编辑PWA的HTML文件（通常是index.html或其他主入口文件）。  
3. **（推荐）apple-touch-icon**：在您的HTML的\<head\>中添加 \<link rel="apple-touch-icon" href="/path/to/your-app-icon.png"\>，以确保添加到主屏幕的图标显示正确。

## **三、 操作步骤**

我们将使用一个名为 pwa-install-prompt 的第三方JavaScript库作为示例，它能自动检测iOS设备并显示相应的安装指南。

### **步骤 1：在您的HTML文件中引入库脚本**

打开您的PWA应用的主HTML文件（例如 index.html）。在 \<head\> 标签的末尾或 \<body\> 标签的开头，添加以下 \<script\> 标签：

\<\!-- 引入 pwa-install-prompt 脚本 \--\>  
\<script async id="wepp-install-modal" src="https://cdn.jsdelivr.net/gh/ryxxn/pwa-install-prompt@main/index.js"\>\</script\>

* async 属性表示脚本将异步加载，不会阻塞页面的解析。  
* id="wepp-install-modal" 是该脚本的标识符。  
* src 指向库文件的CDN地址。

### **步骤 2：理解库的工作原理**

一旦脚本被引入，它会自动执行以下操作：

* **设备检测**：它会检测用户当前使用的设备是否是iOS设备（iPhone、iPad、iPod）。  
* **条件判断**：如果检测到是iOS设备，并且应用尚未添加到主屏幕，它将显示一个自定义的模态弹窗。  
* **显示提示**：这个弹窗会包含图文说明，指导用户如何通过Safari浏览器的“分享”菜单手动将PWA添加到主屏幕。对于非iOS设备，如果支持原生安装提示，它可能会触发原生提示或不显示任何内容（取决于库的具体实现和PWA的配置）。

### **步骤 3：自定义提示内容（可选但推荐）**

虽然 pwa-install-prompt 默认提供英文提示，但您可能希望将其定制为中文或其他语言，或者修改其显示逻辑（例如，在用户访问几次后才显示）。

* **多语言支持**：对于中文提示，您可以尝试使用该库提供的中文版本（如果可用），例如：  
  \<script async id="wepp-install-modal" src="https://cdn.jsdelivr.net/gh/ryxxn/pwa-install-prompt@main/ko/index.js"\>\</script\>

  请注意，这里的 ko 通常代表韩语，您可能需要查找该库是否有官方的中文版本（zh 或 zh-CN）。如果官方没有，您可能需要考虑使用其他支持中文的库，或者自行修改库的源码或通过配置项来注入中文文本（如果库支持）。  
* **高级定制**：对于更复杂的定制（例如，控制显示时机、修改CSS样式），您需要查阅您所选库的官方文档。不同的库提供不同的API和配置选项。

### **步骤 4：保存并部署您的PWA**

保存您修改后的HTML文件，并将其重新部署到您的服务器上。确保您的PWA的所有文件（包括manifest.json和Service Worker）都已更新并可访问。

## **四、 验证**

1. **在iOS设备上测试**：  
   * 使用Safari浏览器打开您的PWA应用的URL。  
   * 浏览您的应用。在满足库设定的显示条件时，应该会弹出一个模态窗口，提示您如何将应用添加到主屏幕。  
   * 按照提示中的步骤，尝试将应用添加到主屏幕。成功后，您的应用图标应该会出现在主屏幕上。  
2. **在非iOS设备上测试**：  
   * 在Android Chrome或Windows Chrome上打开您的PWA。  
   * 确认PWA的安装提示（如果您的应用有自定义触发，则触发您的自定义按钮）是否仍能正常工作，且iOS的自定义提示不会干扰。

## **五、 重要注意事项**

* **iOS的限制**：再次强调，iOS系统本身不提供 beforeinstallprompt 事件来触发原生的PWA安装提示。所有在iOS上看到的“提示安装”都是通过JavaScript和CSS实现的自定义UI。  
* **用户体验**：确保您的自定义提示清晰、简洁，并提供明确的图文指导，因为用户需要手动操作。  
* **兼容性**：在选择第三方库时，请务必查看其文档，了解其兼容性、定制选项和维护状态。

通过遵循这些步骤，您应该能够成功地在您的PWA中为iOS用户提供一个友好的安装引导。