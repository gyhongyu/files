# Files | Gemini API | Google AI for Developers

> **Note:** 本文档是对 https://ai.google.dev/api/files 页面的整理与 Markdown 格式转换，其中内容与官网同步更新可能存在延迟。建议重要开发时查阅[官方文档](https://ai.google.dev/api/files)。

## 文件 API

Gemini API 支持与媒体文件（图片、音频、视频、文本、PDF等）交互。你可以将本地文件上传到 Gemini Files API，获取资源 file.uri，之后作为 prompt 的一部分传递，或在多轮对话、批量请求中复用。

相关指南：[Prompting with media](https://ai.google.dev/docs/prompt-media)

---

## 文件资源

一个文件资源表示上传到 Gemini 的媒体文件。你可以在多个请求或长会话中复用相同文件，上传接口支持多种内容类型。

#### 文件对象结构示例

```
{
  "name": "files/AH1a2Bc3DEFghi...",
  "display_name": "photo.png",
  "mime_type": "image/png",
  "state": "ACTIVE",
  "file_size_bytes": 31415,
  "create_time": "2024-03-20T17:05:54.779318933Z",
  "update_time": "2024-03-20T17:06:05.157445Z",
  "uri": "gs://...",
}
```

### 字段说明

| 字段              | 类型     | 说明                                  |
|-------------------|----------|-------------------------------------|
| name              | string   | 文件资源唯一标识符（如 `files/xxx...`）|
| display_name      | string   | 文件的显示名（可选，通常为上传文件名） |
| mime_type         | string   | MIME 类型，如 image/png 或 audio/mp3 |
| state             | enum     | 文件状态，见下表                     |
| file_size_bytes   | int      | 文件大小，单位：字节                 |
| create_time       | string   | ISO 8601 格式创建时间                 |
| update_time       | string   | ISO 8601 格式最后更新时间             |
| uri               | string   | 用于 prompt 的文件 URI               |

#### 状态说明 `state`

| 状态         | 说明                                                  |
|--------------|-----------------------------------------------------|
| ACTIVE       | 文件已可用                                           |
| CREATING     | 文件上传中/处理中                                   |
| DELETING     | 文件正在被删除                                      |
| FAILED       | 文件处理失败                                        |
| PROCESSING   | 文件正在处理（如大视频/大PDF解析）                 |

---

## 文件上传

### media.upload

允许你将二进制媒体（图片、音频、视频、文本、PDF等）上传至 Gemini Files API。

**HTTP 请求**

```
POST https://generativelanguage.googleapis.com/v1beta/files?key=API_KEY
Content-Type: multipart/form-data
```

请求采用 [Resumable Uploads](https://cloud.google.com/storage/docs/performing-resumable-uploads)。

### 请求参数

| 参数          | 必须 | 类型   | 说明                                |
|---------------|------|--------|-------------------------------------|
| file          | 是   | bytes  | 媒体文件本体                        |
| display_name  | 否   | string | 文件显示名，默认为原文件名           |
| mime_type     | 是   | string | 文件 MIME 类型，如 image/png        |

**文件大小/限制**

- 单文件最大 5GB
- 受限于 API 配额及账户配额

---

## 文件获取

### files.get

根据 name 获取文件的元数据信息和状态。

**HTTP 请求**

```
GET https://generativelanguage.googleapis.com/v1beta/files/{name}
```

---

## 文件删除

### files.delete

即刻删除指定 name 的文件资源。

**HTTP 请求**

```
DELETE https://generativelanguage.googleapis.com/v1beta/files/{name}
```

---

## 文件列举

### files.list

获取你账户下所有文件资源元数据，列出所有已上传文件和状态。

**HTTP 请求**

```
GET https://generativelanguage.googleapis.com/v1beta/files
```

---

## 示例代码

以下分别是图片、音频、文本、视频、PDF等类型的上传及后续使用示例，覆盖 Python, Node.js, Go, Shell。

### 图片 (Image)

#### Python

```
from google import genai

client = genai.Client()
myfile = client.files.upload(file=media / "Cajun_instruments.jpg")
print(f"{myfile=}")

result = client.models.generate_content(
    model="gemini-2.0-flash",
    contents=[
        myfile,
        "\n\n",
        "Can you tell me about the instruments in this photo?",
    ],
)
print(f"{result.text=}")
```

#### Node.js

```
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

const myfile = await ai.files.upload({
    file: path.join(media, "Cajun_instruments.jpg"),
    config: { mimeType: "image/jpeg" },
});
console.log("Uploaded file:", myfile);

const result = await ai.models.generateContent({
    model: "gemini-2.0-flash",
    contents: createUserContent([
        createPartFromUri(myfile.uri, myfile.mimeType),
        "\n\n",
        "Can you tell me about the instruments in this photo?",
    ]),
});
console.log("result.text=", result.text);
```

#### Go

```
ctx := context.Background()
client, err := genai.NewClient(ctx, &genai.ClientConfig{
    APIKey: os.Getenv("GEMINI_API_KEY"),
    Backend: genai.BackendGeminiAPI,
})
if err != nil { log.Fatal(err) }

myfile, err := client.Files.UploadFromPath(
    ctx,
    filepath.Join(getMedia(), "Cajun_instruments.jpg"),
    &genai.UploadFileConfig{ MIMEType : "image/jpeg", },
)
if err != nil { log.Fatal(err) }

fmt.Printf("myfile=%+v\n", myfile)
parts := []*genai.Part{
    genai.NewPartFromURI(myfile.URI, myfile.MIMEType),
    genai.NewPartFromText("\n\n"),
    genai.NewPartFromText("Can you tell me about the instruments in this photo?"),
}
contents := []*genai.Content{
    genai.NewContentFromParts(parts, genai.RoleUser),
}
response, err := client.Models.GenerateContent(ctx, "gemini-2.0-flash", contents, nil)
if err != nil { log.Fatal(err) }
text := response.Text()
fmt.Printf("result.text=%s\n", text)
```

#### Shell

```
MIME_TYPE=$(file -b --mime-type "${IMG_PATH_2}")
NUM_BYTES=$(wc -c < "${IMG_PATH_2}")
DISPLAY_NAME=TEXT
tmp_header_file=upload-header.tmp

# Initial resumable request defining metadata.
curl "${BASE_URL}/upload/v1beta/files?key=${GEMINI_API_KEY}" \
    -D upload-header.tmp \
    -H "X-Goog-Upload-Protocol: resumable" \
    -H "X-Goog-Upload-Command: start" \
    -H "X-Goog-Upload-Header-Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Header-Content-Type: ${MIME_TYPE}" \
    -H "Content-Type: application/json" \
    -d "{'file': {'display_name': '${DISPLAY_NAME}'}}" 2> /dev/null

upload_url=$(grep -i "x-goog-upload-url: " "${tmp_header_file}" | cut -d" " -f2 | tr -d "\r")
rm "${tmp_header_file}"

curl "${upload_url}" \
    -H "Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Offset: 0" \
    -H "X-Goog-Upload-Command: upload, finalize" \
    --data-binary "@${IMG_PATH_2}" 2> /dev/null > file_info.json

file_uri=$(jq ".file.uri" file_info.json)
echo file_uri=$file_uri

curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=$GEMINI_API_KEY" \
    -H 'Content-Type: application/json' \
    -X POST \
    -d '{
        "contents": [{
            "parts":[
                {"text": "Can you tell me about the instruments in this photo?"},
                {"file_data": {"mime_type": "image/jpeg", "file_uri": '$file_uri'}}
            ]
        }]
    }' 2> /dev/null > response.json

cat response.json
jq ".candidates[].content.parts[].text" response.json
```

### 音频 (Audio)

#### Python

```
from google import genai

client = genai.Client()
myfile = client.files.upload(file=media / "sample.mp3")
print(f"{myfile=}")

result = client.models.generate_content(
    model="gemini-2.0-flash", contents=[myfile, "Describe this audio clip"]
)
print(f"{result.text=}")
```

#### Node.js

```
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

const myfile = await ai.files.upload({
    file: path.join(media, "sample.mp3"),
    config: { mimeType: "audio/mpeg" },
});
console.log("Uploaded file:", myfile);

const result = await ai.models.generateContent({
    model: "gemini-2.0-flash",
    contents: createUserContent([
        createPartFromUri(myfile.uri, myfile.mimeType),
        "Describe this audio clip",
    ]),
});
console.log("result.text=", result.text);
```

#### Go

```
ctx := context.Background()
client, err := genai.NewClient(ctx, &genai.ClientConfig{
    APIKey: os.Getenv("GEMINI_API_KEY"),
    Backend: genai.BackendGeminiAPI,
})
if err != nil { log.Fatal(err) }

myfile, err := client.Files.UploadFromPath(
    ctx,
    filepath.Join(getMedia(), "sample.mp3"),
    &genai.UploadFileConfig{ MIMEType : "audio/mpeg", },
)
if err != nil { log.Fatal(err) }

fmt.Printf("myfile=%+v\n", myfile)
parts := []*genai.Part{
    genai.NewPartFromURI(myfile.URI, myfile.MIMEType),
    genai.NewPartFromText("Describe this audio clip"),
}
contents := []*genai.Content{
    genai.NewContentFromParts(parts, genai.RoleUser),
}
response, err := client.Models.GenerateContent(ctx, "gemini-2.0-flash", contents, nil)
if err != nil { log.Fatal(err) }
text := response.Text()
fmt.Printf("result.text=%s\n", text)
```

#### Shell

```
MIME_TYPE=$(file -b --mime-type "${AUDIO_PATH}")
NUM_BYTES=$(wc -c < "${AUDIO_PATH}")
DISPLAY_NAME=AUDIO
tmp_header_file=upload-header.tmp

curl "${BASE_URL}/upload/v1beta/files?key=${GEMINI_API_KEY}" \
    -D upload-header.tmp \
    -H "X-Goog-Upload-Protocol: resumable" \
    -H "X-Goog-Upload-Command: start" \
    -H "X-Goog-Upload-Header-Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Header-Content-Type: ${MIME_TYPE}" \
    -H "Content-Type: application/json" \
    -d "{'file': {'display_name': '${DISPLAY_NAME}'}}" 2> /dev/null

upload_url=$(grep -i "x-goog-upload-url: " "${tmp_header_file}" | cut -d" " -f2 | tr -d "\r")
rm "${tmp_header_file}"

curl "${upload_url}" \
    -H "Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Offset: 0" \
    -H "X-Goog-Upload-Command: upload, finalize" \
    --data-binary "@${AUDIO_PATH}" 2> /dev/null > file_info.json

file_uri=$(jq ".file.uri" file_info.json)
echo file_uri=$file_uri

curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=$GEMINI_API_KEY" \
    -H 'Content-Type: application/json' \
    -X POST \
    -d '{
        "contents": [{
            "parts":[
                {"text": "Describe this audio clip"},
                {"file_data":{"mime_type": "audio/mp3", "file_uri": '$file_uri'}}
            ]
        }]
    }' 2> /dev/null > response.json

cat response.json
jq ".candidates[].content.parts[].text" response.json
```

### 文本 (Text)

#### Python

```
from google import genai

client = genai.Client()
myfile = client.files.upload(file=media / "poem.txt")
print(f"{myfile=}")

result = client.models.generate_content(
    model="gemini-2.0-flash",
    contents=[myfile, "\n\n", "Can you add a few more lines to this poem?"],
)
print(f"{result.text=}")
```

#### Node.js

```
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

const myfile = await ai.files.upload({
    file: path.join(media, "poem.txt"),
});
console.log("Uploaded file:", myfile);

const result = await ai.models.generateContent({
    model: "gemini-2.0-flash",
    contents: createUserContent([
        createPartFromUri(myfile.uri, myfile.mimeType),
        "\n\n",
        "Can you add a few more lines to this poem?",
    ]),
});
console.log("result.text=", result.text);
```

#### Go

```
ctx := context.Background()
client, err := genai.NewClient(ctx, &genai.ClientConfig{
    APIKey: os.Getenv("GEMINI_API_KEY"),
    Backend: genai.BackendGeminiAPI,
})
if err != nil { log.Fatal(err) }

myfile, err := client.Files.UploadFromPath(
    ctx,
    filepath.Join(getMedia(), "poem.txt"),
    &genai.UploadFileConfig{ MIMEType : "text/plain", },
)
if err != nil { log.Fatal(err) }

fmt.Printf("myfile=%+v\n", myfile)
parts := []*genai.Part{
    genai.NewPartFromURI(myfile.URI, myfile.MIMEType),
    genai.NewPartFromText("\n\n"),
    genai.NewPartFromText("Can you add a few more lines to this poem?"),
}
contents := []*genai.Content{
    genai.NewContentFromParts(parts, genai.RoleUser),
}
response, err := client.Models.GenerateContent(ctx, "gemini-2.0-flash", contents, nil)
if err != nil { log.Fatal(err) }
text := response.Text()
fmt.Printf("result.text=%s\n", text)
```

#### Shell

```
MIME_TYPE=$(file -b --mime-type "${TEXT_PATH}")
NUM_BYTES=$(wc -c < "${TEXT_PATH}")
DISPLAY_NAME=TEXT
tmp_header_file=upload-header.tmp

curl "${BASE_URL}/upload/v1beta/files?key=${GEMINI_API_KEY}" \
    -D upload-header.tmp \
    -H "X-Goog-Upload-Protocol: resumable" \
    -H "X-Goog-Upload-Command: start" \
    -H "X-Goog-Upload-Header-Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Header-Content-Type: ${MIME_TYPE}" \
    -H "Content-Type: application/json" \
    -d "{'file': {'display_name': '${DISPLAY_NAME}'}}" 2> /dev/null

upload_url=$(grep -i "x-goog-upload-url: " "${tmp_header_file}" | cut -d" " -f2 | tr -d "\r")
rm "${tmp_header_file}"

curl "${upload_url}" \
    -H "Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Offset: 0" \
    -H "X-Goog-Upload-Command: upload, finalize" \
    --data-binary "@${TEXT_PATH}" 2> /dev/null > file_info.json

file_uri=$(jq ".file.uri" file_info.json)
echo file_uri=$file_uri

curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=$GEMINI_API_KEY" \
    -H 'Content-Type: application/json' \
    -X POST \
    -d '{
        "contents": [{
            "parts":[
                {"text": "Can you add a few more lines to this poem?"},
                {"file_data":{"mime_type": "text/plain", "file_uri": '$file_uri'}}
            ]
        }]
    }' 2> /dev/null > response.json

cat response.json
jq ".candidates[].content.parts[].text" response.json

name=$(jq ".file.name" file_info.json)
curl https://generativelanguage.googleapis.com/v1beta/files/$name > file_info.json
name=$(jq ".file.name" file_info.json)
echo name=$name
file_uri=$(jq ".file.uri" file_info.json)
echo file_uri=$file_uri

curl --request "DELETE" https://generativelanguage.googleapis.com/v1beta/files/$name?key=$GEMINI_API_KEY
```

### 视频 (Video)

#### Python

```
from google import genai
import time

client = genai.Client()
myfile = client.files.upload(file=media / "Big_Buck_Bunny.mp4")
print(f"{myfile=}")

while not myfile.state or myfile.state.name != "ACTIVE":
    print("Processing video...")
    print("File state:", myfile.state)
    time.sleep(5)
    myfile = client.files.get(name=myfile.name)

result = client.models.generate_content(
    model="gemini-2.0-flash", contents=[myfile, "Describe this video clip"]
)
print(f"{result.text=}")
```

#### Node.js

```
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

let myfile = await ai.files.upload({
    file: path.join(media, "Big_Buck_Bunny.mp4"),
    config: { mimeType: "video/mp4" },
});
console.log("Uploaded video file:", myfile);

while (!myfile.state || myfile.state.toString() !== "ACTIVE") {
    console.log("Processing video...");
    console.log("File state: ", myfile.state);
    await sleep(5000);
    myfile = await ai.files.get({ name: myfile.name });
}

const result = await ai.models.generateContent({
    model: "gemini-2.0-flash",
    contents: createUserContent([
        createPartFromUri(myfile.uri, myfile.mimeType),
        "Describe this video clip",
    ]),
});
console.log("result.text=", result.text);
```

#### Go

```
ctx := context.Background()
client, err := genai.NewClient(ctx, &genai.ClientConfig{
    APIKey: os.Getenv("GEMINI_API_KEY"),
    Backend: genai.BackendGeminiAPI,
})
if err != nil { log.Fatal(err) }

myfile, err := client.Files.UploadFromPath(
    ctx,
    filepath.Join(getMedia(), "Big_Buck_Bunny.mp4"),
    &genai.UploadFileConfig{ MIMEType : "video/mp4", },
)
if err != nil { log.Fatal(err) }
fmt.Printf("myfile=%+v\n", myfile)
for myfile.State == genai.FileStateUnspecified || myfile.State != genai.FileStateActive {
    fmt.Println("Processing video...")
    fmt.Println("File state:", myfile.State)
    time.Sleep(5 * time.Second)
    myfile, err = client.Files.Get(ctx, myfile.Name, nil)
    if err != nil { log.Fatal(err) }
}
parts := []*genai.Part{
    genai.NewPartFromURI(myfile.URI, myfile.MIMEType),
    genai.NewPartFromText("Describe this video clip"),
}
contents := []*genai.Content{
    genai.NewContentFromParts(parts, genai.RoleUser),
}

response, err := client.Models.GenerateContent(ctx, "gemini-2.0-flash", contents, nil)
if err != nil { log.Fatal(err) }
text := response.Text()
fmt.Printf("result.text=%s\n", text)
```

#### Shell

```
MIME_TYPE=$(file -b --mime-type "${VIDEO_PATH}")
NUM_BYTES=$(wc -c < "${VIDEO_PATH}")
DISPLAY_NAME=VIDEO_PATH

curl "${BASE_URL}/upload/v1beta/files?key=${GEMINI_API_KEY}" \
    -D upload-header.tmp \
    -H "X-Goog-Upload-Protocol: resumable" \
    -H "X-Goog-Upload-Command: start" \
    -H "X-Goog-Upload-Header-Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Header-Content-Type: ${MIME_TYPE}" \
    -H "Content-Type: application/json" \
    -d "{'file': {'display_name': '${DISPLAY_NAME}'}}" 2> /dev/null

upload_url=$(grep -i "x-goog-upload-url: " "${tmp_header_file}" | cut -d" " -f2 | tr -d "\r")
rm "${tmp_header_file}"

curl "${upload_url}" \
    -H "Content-Length: ${NUM_BYTES}" \
    -H "X-Goog-Upload-Offset: 0" \
    -H "X-Goog-Upload-Command: upload, finalize" \
    --data-binary "@${VIDEO_PATH}" 2> /dev/null > file_info.json

file_uri=$(jq ".file.uri" file_info.json)
echo file_uri=$file_uri
state=$(jq ".file.state" file_info.json)
echo state=$state

# 等待状态为 ACTIVE
while [[ "$state" != '"ACTIVE"' ]]; do
    echo "Processing video..."
    sleep 5
    name=$(jq ".file.name" file_info.json)
    curl https://generativelanguage.googleapis.com/v1beta/files/$name > file_info.json
    state=$(jq ".file.state" file_info.json)
done

curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=$GEMINI_API_KEY" \
    -H 'Content-Type: application/json' \
    -X POST \
    -d '{
        "contents": [{
            "parts":[
                {"text": "Describe this video clip"},
                {"file_data":{"mime_type": "video/mp4", "file_uri": '$file_uri'}}
            ]
        }]
    }' 2> /dev/null > response.json

cat response.json
jq ".candidates[].content.parts[].text" response.json
```

### PDF

> 官方文档目前仅简要说明。例如上传 PDF 文档后，可以通过 prompt 请求 Gemini 对 PDF 内容进行总结、问答等。

#### Python

```
from google import genai

client = genai.Client()
myfile = client.files.upload(file=media / "document.pdf")
print(f"{myfile=}")

result = client.models.generate_content(
    model="gemini-2.0-pro",
    contents=[myfile, "Summarize this document"],
)
print(f"{result.text=}")
```

---

## 最佳实践提示

- **文件复用**：同一个文件可用于多个 prompt 调用，也可多轮会话共享，减少重复上传。
- **处理耗时说明**：大文件（PDF、视频）上传完后，建议轮询状态为 ACTIVE 后再用于推理。
- **文件删除**：不再需要时请主动删除无用文件释放存储。
- **权限管理**：确保你的 API Key 对应的账户有文件管理权限。
- **多语言客户端支持**：Google 提供 Python、Node.js、Go 等客户端库，也支持 RESTful API 调用。

## 参考资源

- [Prompting with media](https://ai.google.dev/docs/prompt-media)
- [Gemini API 官方文档](https://ai.google.dev/)
- [Google Cloud Storage Resumable Upload](https://cloud.google.com/storage/docs/performing-resumable-uploads)

---
```

如需自动化用，可复制以上代码块，粘贴到`.md`文件即可直接使用。

如果你需要PDF等其它媒体类型更详细的 SDK 示例，或有内容遗漏，请告诉我！
