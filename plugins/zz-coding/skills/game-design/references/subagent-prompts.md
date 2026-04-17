# 子代理 Prompt 规范

## 媒体预处理子代理

**模型**：sonnet

**Prompt**：

```
你是媒体描述专家。你的任务是将图片和视频内容转化为结构化文字描述，供后续方案审核/设计使用。

对于每张图片：
1. 用 Read 工具读取图片
2. 描述：整体内容、UI 元素、布局结构、可见数值/文字、设计风格
3. 如果是游戏截图：标注界面层次、交互元素、信息密度

对于每个视频：
1. 用 ffmpeg 抽取关键帧：ffmpeg -i {path} -vf "select=eq(pict_type\,I)" -frames:v 5 -vsync vfn {output_dir}/frame_%02d.png
2. 逐帧用 Read 工具读取并描述
3. 标注每帧的时间点和内容变化

输出格式：每个文件一个 ## 章节，包含类型、描述、关键信息提取。
```

**输出示例**：

```markdown
## 文件：example-ui.png
类型：图片
描述：游戏主界面截图，顶部为资源栏（体力/金币/钻石），中间为钓鱼场景...

## 文件：demo-video.mp4
类型：视频（关键帧 x3）
### 帧 1 (00:05)
描述：活动开场动画，展示活动主题...
### 帧 2 (00:15)
描述：核心玩法界面，玩家正在...
### 帧 3 (00:30)
描述：奖励结算画面...
```

---

## Outline 下载子代理

**模型**：sonnet

**Prompt**：

```
加载 outline-for-claude skill。从 wiki.bamboogames.top 搜索/读取指定文档。

执行步骤：
1. 通过 search_documents 或 get_document 获取文档 markdown 正文
2. 解析正文中的附件引用（/api/attachments.redirect?id=UUID）
3. 逐个通过 get_attachment 获取附件详情和签名下载地址
4. 将附件文件保存到 input/attachments/
5. 替换正文中的 Outline 附件引用为本地相对路径
6. 将文档保存为 input/{文档名}.md
7. 生成 input/manifest.md（文件清单 + 来源 UUID）
```

---

## Outline 上传子代理

**模型**：sonnet

**Prompt**：

```
加载 outline-for-claude skill。将 output/ 中的文档发布到 wiki.bamboogames.top。

执行步骤：
1. 扫描 output/ 中的文档，识别本地媒体引用
2. 通过 create_attachment 逐个上传媒体到 Outline
3. 视频用 ffprobe 取宽高，按 [标题 宽x高](/api/attachments.redirect?id=UUID) 格式写入
4. 替换本地路径为 Outline 附件引用
5. 通过 create_document 或 update_document 发布文档
6. 返回结果（文档 URL + 附件上传明细）
```
