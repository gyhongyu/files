### **技术设计文档：PPTX保真格式AI翻译插件 (Gemini-Powered)**

| 文档版本 | 1.0 |
| :---- | :---- |
| **日期** | 2025年8月1日 |
| **作者** | \[您的AI架构师\] |
| **状态** | 定稿 |
| **审批人** | 项目负责人 |

#### **1.0 执行摘要 (Executive Summary)**

本项目旨在开发一款企业级的PowerPoint插件，其核心功能是提供高保真度的幻灯片文本翻译。传统翻译插件在处理复杂格式（如混合字体、颜色、粗斜体、项目符号等）时，常导致格式丢失。本方案将利用Google Gemini Pro Vision多模态模型的独特能力，通过\*\*“文本结构化+视觉上下文”\*\*的组合输入，让AI不仅翻译内容，更能理解并重构原始格式，从而实现近乎100%的格式保真度。

该系统架构分为三层：前端PowerPoint插件（客户端）、后端BFF服务（业务流程编排）以及后端的Google Cloud AI服务。此设计确保了系统的安全性、可扩展性和可维护性。

#### **2.0 系统架构 (System Architecture)**

我们将采用经典的三层架构模型：**Client \-\> BFF \-\> AI Service**。

graph TD  
    subgraph "用户设备"  
        A\[PowerPoint Add-in (Client)\]  
    end

    subgraph "云端 (Google Cloud Platform)"  
        B\[Backend For Frontend (BFF)\]  
        C\[Google Cloud Storage\]  
        D\[Google Gemini API\]  
        E\[Google Secret Manager\]  
    end

    A \-- 1\. 提取内容与截图, 发起翻译请求 (HTTPS POST) \--\> B  
    B \-- 2\. 上传截图 \--\> C  
    C \-- 3\. 返回图片URL \--\> B  
    B \-- 4\. 从Secret Manager获取API Key \--\> E  
    E \-- 5\. 返回API Key \--\> B  
    B \-- 6\. 构造多模态Prompt, 调用Gemini API \--\> D  
    D \-- 7\. 返回结构化翻译结果 \--\> B  
    B \-- 8\. 将结果返回给客户端 \--\> A

    style A fill:\#cce5ff,stroke:\#333,stroke-width:2px  
    style B fill:\#d5e8d4,stroke:\#333,stroke-width:2px  
    style D fill:\#f8cecc,stroke:\#333,stroke-width:2px

**组件职责:**

* **PowerPoint Add-in (Client):**  
  * **技术栈:** Office.js, TypeScript, React (推荐，用于复杂UI)。  
  * **职责:**  
    1. 提供用户交互界面（任务窗格）。  
    2. 使用 Office.js API 提取选定形状(Shape)的详细格式化文本数据，并将其序列化为JSON。  
    3. 使用 Office.js document.getFileAsync API 获取当前幻灯片的截图。  
    4. 将序列化的文本JSON和截图安全地发送到BFF。  
    5. 接收来自BFF的结构化翻译结果。  
    6. 精确地将结果渲染回PowerPoint，逐个Run重建格式。  
* **Backend For Frontend (BFF):**  
  * **技术栈:** Node.js, TypeScript, Express.js (或Python/FastAPI)，部署于Google Cloud Run。  
  * **职责:**  
    1. 作为客户端的唯一API网关，增加安全性。  
    2. 管理和安全存储Google Cloud凭证和Gemini API密钥（通过Secret Manager）。  
    3. 接收客户端请求，处理截图（上传至GCS并生成临时URL）。  
    4. **核心职责：Prompt工程**。将文本JSON和截图URL整合成一个复杂的多模态Prompt。  
    5. 调用Gemini API，处理其响应。  
    6. 将Gemini返回的结构化数据直接透传给客户端。  
* **Google Cloud Services:**  
  * **Gemini API:** 核心AI引擎。我们将使用 gemini-1.5-pro 或 gemini-pro-vision 模型。  
  * **Cloud Storage:** 临时存储幻灯片截图，供Gemini API访问。  
  * **Secret Manager:** 安全存储所有服务密钥。  
  * **Cloud Run:** 托管无状态、可自动伸缩的BFF服务。

#### **3.0 核心算法：格式化文本的序列化、翻译与重构**

这是本项目的技术核心。我们将此过程分解为四个步骤。

**Step 1: 客户端数据提取与序列化 (Extract & Serialize)**

当用户点击“翻译”时，插件将对选中的每个Shape执行以下操作：

* **遍历段落 (Paragraphs):**  
* **遍历文本运行 (Runs):** Run是具有统一格式的最小文本单元。  
* **提取属性:** 对于每个Run，提取其 font 对象的全部属性：name (字体), size, color, bold, italic, underline 等。  
* **构建JSON对象:** 将所有信息序列化为一个定义好的 ShapeContent JSON对象。

**Step 2: 截图生成与请求封装**

* 客户端使用 Office.js 的 document.getFileAsync 方法，fileType: Office.FileType.Png 来获取当前幻灯片的高清截图。  
* 客户端将ShapeContent JSON和图片文件通过 multipart/form-data POST请求发送到BFF。

**Step 3: BFF的Prompt工程与Gemini调用**

BFF收到请求后：

1. 将图片上传到GCS，获得一个可公开访问的临时URL。  
2. 构造一个精密的**多模态Prompt**。这是成功的关键。

**示例Prompt:**

You are an expert PowerPoint format analysis and translation AI. Your task is to translate a given text while perfectly preserving its original formatting structure.

I will provide you with two pieces of information:  
1\. A JSON object (\`original\_content\`) detailing the text and its rich-text formatting, paragraph by paragraph, and run by run.  
2\. A URL to a screenshot (\`visual\_context\_image\`) of the slide for visual reference, which helps understand layout, emphasis, and context that may not be fully captured in the JSON.

Your task:  
1\. Analyze both the JSON structure and the visual context from the image.  
2\. Translate the text content within the JSON from the source language to the target language: \[目标语言\].  
3\. Return a new JSON object in the \*\*exact same schema\*\* as the input \`original\_content\` object, but with the \`text\` fields translated.  
4\. \*\*CRITICAL:\*\* Do not alter the structure. The number of paragraphs and the number of runs within each paragraph must match the original, unless merging/splitting is logically necessary for the target language grammar. All formatting properties (\`font\`, \`color\`, \`bold\`, etc.) must be preserved and correctly mapped to the translated text segments. If a run was bold, its translated counterpart must also be bold.

Here is the data:

\*\*\[Visual Context Image URL\]:\*\*  
\[此处插入GCS的图片URL\]

\*\*\[Original Content JSON\]:\*\*  
\[此处插入客户端生成的ShapeContent JSON字符串\]

\*\*\[Target Language\]:\*\*  
ZH-CN

Now, generate the translated JSON object.

**Step 4: 客户端接收与渲染 (Receive & Render)**

1. BFF将Gemini返回的、符合预定Schema的JSON字符串直接返回给客户端。  
2. 客户端解析这个新的JSON对象 TranslatedShapeContent。  
3. **清空**目标Shape中的原有文本 (shape.textFrame.textRange.clear())。  
4. **遍历** TranslatedShapeContent 的paragraphs和runs。  
5. 对于每一个run，使用 textRange.insertText() 方法插入翻译后的文本，然后**立即**对其font对象应用JSON中指定的全部格式属性 (color, size, bold等)。  
6. 这个过程相当于用AI返回的“蓝图”在PowerPoint中“重新绘制”文本，从而达到100%的格式保真。

#### **4.0 API接口定义 (BFF Endpoint)**

* **Endpoint:** POST /api/v1/translate/shape  
* **Request Type:** multipart/form-data  
* **Form Fields:**  
  * shape\_content: (string) 序列化后的 ShapeContent JSON对象。  
  * image: (file) 幻灯片截图PNG文件。  
  * target\_language: (string) e.g., "EN-US", "ZH-CN".  
* **Success Response (200 OK):**  
  {  
    "status": "success",  
    "data": {  
      // Gemini返回的、结构化的TranslatedShapeContent JSON对象  
    }  
  }

* **Error Response (e.g., 400, 500):**  
  {  
    "status": "error",  
    "message": "Descriptive error message."  
  }

#### **5.0 数据结构定义 (Data Structures)**

**ShapeContent / TranslatedShapeContent JSON Schema:**

// 定义了最小文本单元的接口  
interface ITextRun {  
  text: string;  
  font: {  
    name: string;      // e.g., "Calibri"  
    size: number;      // e.g., 18  
    color: string;     // e.g., "\#FF0000"  
    bold: boolean;  
    italic: boolean;  
    underline: Office.MailMerge.Underline; // "None", "Single", etc.  
  };  
}

// 定义了段落的接口  
interface IParagraph {  
  runs: ITextRun\[\];  
  // 未来可扩展段落级别格式，如对齐、缩进等  
  // alignment: Office.MailMerge.Alignment;  
}

// 顶层对象接口  
interface IShapeContent {  
  id: string; // shape.id  
  paragraphs: IParagraph\[\];  
}

#### **6.0 安全、性能与风险**

* **安全 (Security):**  
  * **凭证管理:** 所有API密钥和服务账户凭证必须存储在Google Secret Manager中，BFF在运行时动态获取。严禁硬编码在代码或配置文件中。  
  * **通信安全:** 客户端与BFF、BFF与Google服务之间的所有通信必须强制使用TLS 1.2+。  
  * **数据隐私:** 截图和文本内容在GCS中应设置生命周期规则，在处理完毕后（例如24小时内）自动删除，确保用户数据不被长期保留。  
* **性能 (Performance):**  
  * **客户端响应:** 在等待AI响应期间，插件UI必须显示清晰的加载状态，防止用户误以为程序卡死。  
  * **API延迟:** Gemini Pro Vision的调用会有延迟。BFF可以考虑使用WebSocket或长轮询来处理长时间运行的任务，但这在V1版本中可能过度设计。初版采用简单的异步请求/响应模型即可。  
* **风险与缓解 (Risks & Mitigation):**  
  * **风险1:** Gemini返回的JSON格式损坏或不符合Schema。  
    * **缓解:** BFF层需要增加一个校验步骤。如果返回的JSON无效，则向客户端返回一个可控的错误，并记录此问题以供分析和Prompt调优。  
  * **风险2:** 翻译质量或格式理解错误（“幻觉”）。  
    * **缓解:** 这是LLM的固有风险。通过不断优化Prompt工程、提供高质量的截图和结构化数据，可以将此风险降至最低。提供一个“反馈”按钮让用户报告问题，是持续改进的关键。  
  * **风险3:** API成本失控。  
    * **缓解:** BFF层必须实现严格的日志记录和监控，追踪API调用次数和Token消耗。可为用户或组织设置配额（Quota）管理。

#### **7.0 部署与运维 (Deployment & Operations)**

* **CI/CD:** 建议使用GitHub Actions或Google Cloud Build。  
  * **BFF:** 每次推送到主分支时，自动构建Docker镜像，并部署到Cloud Run。  
  * **Client:** 自动打包前端资源（HTML/JS/CSS）和manifest.xml，并上传到GCS的一个静态网站托管Bucket。manifest.xml中的资源URL将指向此Bucket。  
* **部署:**  
  * **插件清单 (manifest.xml):** 通过Microsoft 365管理员中心的“集中部署”功能，推送给组织内的所有用户。