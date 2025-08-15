### **主题说明**

本 SOP 文档旨在提供一套完整的、可操作的指南，帮助你在本地搭建一个基于 Node.js 的文本嵌入（Text Embedding）环境。我们的目标是使用开源的 @xenova/transformers 库，实现将文本转换为向量数据的能力，作为云服务（如 Gemini-embedding-001）的完全离线和免费替代方案。

### **准备工作**

开始前，请确保你的系统已满足以下要求。

#### **第一步：安装 Node.js 和 npm**

如果你尚未安装，请从 [Node.js 官网](https://nodejs.org/) 下载并安装。建议安装 LTS (长期支持) 版本，确保其版本号不低于 **18.0.0**。

安装完成后，可以在命令行中运行以下命令来验证安装是否成功：

node \-v  
npm \-v

#### **第二步：创建项目并安装依赖**

在你的项目目录下，打开命令行工具（如终端或 PowerShell），执行以下命令来初始化一个新的 Node.js 项目并安装核心依赖库。

\# 初始化项目，生成 package.json 文件  
npm init \-y

\# 安装 @xenova/transformers 库  
npm install @xenova/transformers

#### **第三步：模型自动下载**

@xenova/transformers 库的一大优势是它能自动处理模型下载。当你第一次运行代码时，它会自动从 Hugging Face 模型仓库下载所需的模型文件（例如 Xenova/all-MiniLM-L6-v2），并将其缓存到本地。这意味着你无需手动下载模型文件，也无需进行复杂的部署操作。

### **操作步骤与代码示例**

现在，我们来编写核心代码。创建一个名为 generate\_embeddings.js 的文件，并将以下代码复制进去。

#### **代码示例**

// generate\_embeddings.js

// 导入 @xenova/transformers 库中的 pipeline 函数  
import { pipeline } from '@xenova/transformers';

/\*\*  
 \* 这是一个异步函数，用于生成文本嵌入向量。  
 \*/  
async function main() {  
    console.log('正在加载文本嵌入模型...');

    // 使用 pipeline 创建一个 "feature-extraction" 管道，即文本嵌入  
    // 'Xenova/all-MiniLM-L6-v2' 是一个在 Hugging Face 上广泛使用的模型  
    const embedder \= await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');

    console.log('模型加载成功！');

    // 准备一些用于生成嵌入向量的文本  
    const texts \= \[  
        '机器学习是一种通过数据训练计算机的技术。',  
        '深度学习是机器学习的一个子领域。',  
        '夕阳西下，天空被染成了橘红色。'  
    \];

    console.log('正在为以下文本生成向量：');  
    console.log(texts);

    // 调用 embedder 函数，将文本转换为向量  
    // options:  
    // \- pooling: 'mean' 表示对所有 token 的输出向量求平均值  
    // \- normalize: true 表示将最终向量归一化为单位向量  
    const output \= await embedder(texts, { pooling: 'mean', normalize: true });

    console.log('\\n生成的文本嵌入向量：');

    // output.data 是一个包含所有向量的 Float32Array  
    // 这里我们只打印第一个文本的向量，并将其转换为标准数组以便于查看  
    const firstVector \= Array.from(output.data.slice(0, 384)); // all-MiniLM-L6-v2 的维度是 384  
    console.log(firstVector);

    // 你也可以打印所有向量的形状  
    console.log('\\n生成的向量维度为:', output.dims);  
}

// 执行主函数  
main();

#### **代码详解**

* import { pipeline } from '@xenova/transformers';：这是从库中导入用于构建任务处理器的核心函数。pipeline 函数能够根据你指定的任务（feature-extraction）自动为你配置好一切。  
* const embedder \= await pipeline(...)：这行代码是关键。它创建了一个用于“特征提取”（即文本嵌入）的任务管道。  
  * 'feature-extraction'：告诉 pipeline 我们要做什么。  
  * 'Xenova/all-MiniLM-L6-v2'：指定要使用的模型。这个模型是开源社区中一个非常受欢迎且高效的文本嵌入模型。  
* const texts \= \[...\]：一个包含你想要转换为向量的文本的数组。  
* await embedder(texts, { ... })：调用嵌入器。  
  * pooling: 'mean'：指定池化策略。它会计算所有单词的向量平均值来得到一个句子级的向量。  
  * normalize: true：对最终的向量进行归一化，使其长度为 1。这对于后续的相似度计算（如余弦相似度）非常有用。  
* output.data：embedder 函数返回的结果，其中 data 属性包含了所有文本的嵌入向量数据。  
* output.dims：返回的向量维度。all-MiniLM-L6-v2 的维度是 384。

### **部署与测试**

最后，在命令行中执行你的脚本来运行测试。

node generate\_embeddings.js

当第一次运行时，你可能会看到终端有一些下载模型的进度条。下载完成后，脚本会输出每个文本对应的嵌入向量。