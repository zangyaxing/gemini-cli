# Gemini CLI 历史记录功能深度解析

## 概述

Gemini CLI 实现了一套完整的历史记录系统，包含会话记录、检查点(checkpoint)机制和长期记忆存储。这套系统允许用户保存和恢复对话会话，维持上下文连续性，并提供智能的记忆管理功能。

## 核心架构

历史记录功能基于以下几个核心组件构建：

### 1. 存储层 (Storage Layer)

位置：`packages/core/src/config/storage.ts`

存储层负责管理所有历史记录数据的物理存储位置：

```typescript
export class Storage {
  // 全局Gemini目录：~/.gemini/
  static getGlobalGeminiDir(): string
  
  // 项目临时目录：~/.gemini/tmp/<project_hash>/
  getProjectTempDir(): string
  
  // 历史记录目录：~/.gemini/history/<project_hash>/
  getHistoryDir(): string
  
  // 检查点目录：~/.gemini/tmp/<project_hash>/checkpoints/
  getProjectTempCheckpointsDir(): string
}
```

**存储结构：**
```
~/.gemini/
├── history/
│   └── <project_hash>/          # Git仓库，用于文件版本控制
├── tmp/
│   └── <project_hash>/
│       ├── chats/               # 会话记录JSON文件
│       └── checkpoints/         # 检查点文件
├── memory.md                    # 长期记忆文件
└── settings.json                # 全局设置
```

### 2. 会话记录服务 (ChatRecordingService)

位置：`packages/core/src/services/chatRecordingService.ts`

这是历史记录系统的核心，负责自动记录所有对话交互：

#### 数据结构

```typescript
export interface MessageRecord {
  id: string;
  timestamp: string;
  content: PartListUnion;
  type: 'user' | 'gemini' | 'info' | 'error' | 'warning';
  // Gemini特有字段
  toolCalls?: ToolCallRecord[];
  thoughts?: Array<ThoughtSummary & { timestamp: string }>;
  tokens?: TokensSummary | null;
  model?: string;
}

export interface ConversationRecord {
  sessionId: string;
  projectHash: string;
  startTime: string;
  lastUpdated: string;
  messages: MessageRecord[];
}
```

#### 核心功能

1. **消息记录**：`recordMessage()` - 记录用户和AI的对话
2. **思考过程记录**：`recordThought()` - 记录AI的推理过程
3. **工具调用记录**：`recordToolCalls()` - 记录工具执行详情
4. **令牌使用统计**：`recordMessageTokens()` - 记录API使用量

#### 文件命名规范

会话文件采用时间戳+会话ID的命名方式：
```
session-YYYY-MM-DDTHH-MM-{sessionId前8位}.json
```

例如：`session-2025-11-24T15-30-a1b2c3d4.json`

### 3. 检查点系统 (Checkpoint System)

位置：`packages/core/src/core/logger.ts`

检查点系统提供会话状态的快照功能：

#### 检查点数据结构

```typescript
export interface Checkpoint {
  history: Content[];        // Gemini API格式的对话历史
  authType?: AuthType;      // 认证类型
}
```

#### 核心功能

1. **保存检查点**：`saveCheckpoint(checkpoint, tag)`
2. **加载检查点**：`loadCheckpoint(tag)`
3. **删除检查点**：`deleteCheckpoint(tag)`

#### 文件安全编码

检查点文件名使用URL编码确保安全：
```typescript
export function encodeTagName(str: string): string {
  return encodeURIComponent(str);
}
```

文件路径：`~/.gemini/tmp/<project_hash>/checkpoints/checkpoint-{encodedTag}.json`

### 4. 会话管理工具 (SessionUtils)

位置：`packages/cli/src/utils/sessionUtils.ts`

提供会话发现、选择和恢复功能：

#### 会话信息结构

```typescript
export interface SessionInfo {
  id: string;              // 会话UUID
  file: string;            // 文件名(不含扩展名)
  fileName: string;        // 完整文件名
  startTime: string;       // 开始时间
  messageCount: number;    // 消息数量
  lastUpdated: string;     // 最后更新时间
  displayName: string;     // 显示名称(首条用户消息)
  firstUserMessage: string; // 首条用户消息内容
  isCurrentSession: boolean; // 是否为当前会话
  index: number;           // 显示索引
}
```

#### 核心功能

1. **会话列表**：`listSessions()` - 获取所有可用会话
2. **会话查找**：`findSession(identifier)` - 通过UUID或索引查找
3. **会话解析**：`resolveSession(resumeArg)` - 解析恢复参数
4. **会话搜索**：支持在会话内容中搜索文本

### 5. Git集成服务 (GitService)

位置：`packages/core/src/services/gitService.ts`

为检查点系统提供版本控制支持：

#### 核心功能

1. **影子仓库初始化**：`setupShadowGitRepository()`
2. **文件快照**：`createFileSnapshot(message)`
3. **快照恢复**：`restoreProjectFromSnapshot(commitHash)`

#### 影子仓库特性

- 独立于项目Git仓库
- 专用配置：`user.name = Gemini CLI`, `email = gemini-cli@google.com`
- 禁用GPG签名
- 继承项目的`.gitignore`规则

### 6. 长期记忆工具 (MemoryTool)

位置：`packages/core/src/tools/memoryTool.ts`

管理跨会话的长期记忆存储：

#### 记忆存储格式

记忆存储在`~/.gemini/memory.md`文件中：

```markdown
## Gemini Added Memories
- 用户偏好：喜欢深色主题
- 项目信息：使用React + TypeScript
- 个人信息：名叫张三，是前端开发者
```

#### 核心功能

1. **记忆保存**：`performAddMemoryEntry()`
2. **内容计算**：`computeNewContent()` - 计算新内容
3. **文件读取**：`readMemoryFileContent()`

## 工作流程

### 会话创建流程

1. **初始化阶段**：
   ```typescript
   const chatRecordingService = new ChatRecordingService(config);
   chatRecordingService.initialize(resumedSessionData);
   ```

2. **文件创建**：
   - 生成时间戳文件名
   - 创建初始会话记录
   - 设置会话ID和项目哈希

3. **目录结构**：
   ```
   ~/.gemini/tmp/<project_hash>/chats/
   └── session-2025-11-24T15-30-a1b2c3d4.json
   ```

### 消息记录流程

1. **用户消息**：
   ```typescript
   chatRecordingService.recordMessage({
     type: 'user',
     content: userContent
   });
   ```

2. **AI思考过程**：
   ```typescript
   chatRecordingService.recordThought({
     title: "分析用户需求",
     content: "用户想要了解历史记录功能..."
   });
   ```

3. **AI响应**：
   ```typescript
   chatRecordingService.recordMessage({
     type: 'gemini',
     content: aiResponse,
     model: "gemini-2.5-pro",
     toolCalls: executedTools,
     thoughts: queuedThoughts,
     tokens: tokenUsage
   });
   ```

### 检查点创建流程

1. **保存对话历史**：
   ```typescript
   await logger.saveCheckpoint({
     history: conversationHistory,
     authType: currentAuthType
   }, tag);
   ```

2. **Git快照**（如果启用）：
   ```typescript
   const commitHash = await gitService.createFileSnapshot(
     `Checkpoint: ${tag}`
   );
   ```

### 会话恢复流程

1. **会话选择**：
   ```typescript
   const sessionSelector = new SessionSelector(config);
   const result = await sessionSelector.resolveSession(resumeArg);
   ```

2. **数据加载**：
   ```typescript
   const chatRecordingService = new ChatRecordingService(config);
   chatRecordingService.initialize({
     conversation: result.sessionData,
     filePath: result.sessionPath
   });
   ```

3. **认证验证**：
   ```typescript
   if (checkpoint.authType !== currentAuthType) {
     throw new Error("认证类型不匹配");
   }
   ```

## 配置选项

### 检查点设置

```typescript
export interface CheckpointingSettings {
  enabled?: boolean;  // 是否启用检查点功能
}
```

### 会话保留设置

```typescript
export interface SessionRetentionSettings {
  enabled?: boolean;     // 启用自动清理
  maxAge?: string;       // 最大保存时间 (如"30d", "7d")
  maxCount?: number;     // 最大保存数量
  minRetention?: string; // 最小保留时间 (默认"1d")
}
```

## 安全特性

### 1. 文件名安全编码

检查点标签使用URL编码防止文件系统问题：
```typescript
// "My Checkpoint" → "My%20Checkpoint"
const safeFileName = encodeTagName(userInput);
```

### 2. 数据验证

会话文件包含严格的字段验证：
```typescript
if (!content.sessionId || !content.messages || 
    !Array.isArray(content.messages)) {
  return { fileName: file, sessionInfo: null }; // 标记为损坏
}
```

### 3. 损坏文件处理

系统优雅处理损坏的会话文件：
- 读取失败时标记为损坏
- 在会话列表中过滤掉损坏文件
- 提供错误日志用于调试

## 性能优化

### 1. 延迟加载

会话内容仅在需要时加载：
```typescript
export interface GetSessionOptions {
  includeFullContent?: boolean; // 仅在搜索时加载
}
```

### 2. 内存缓存

```typescript
private cachedLastConvData: string | null = null;

// 仅在内容变化时写入文件
if (this.cachedLastConvData !== JSON.stringify(conversation, null, 2)) {
  fs.writeFileSync(this.conversationFile, newContent);
  this.cachedLastConvData = newContent;
}
```

### 3. 增量更新

工具调用和令牌信息采用增量更新方式：
```typescript
// 更新现有工具调用而非重写整个消息
lastMsg.toolCalls = lastMsg.toolCalls.map(toolCall => {
  const incoming = toolCalls.find(tc => tc.id === toolCall.id);
  return incoming ? { ...toolCall, ...incoming } : toolCall;
});
```

## 集成点

### 1. CLI命令集成

- `/chat` - 创建新会话或恢复现有会话
- `/checkpoint` - 管理检查点
- `--resume` - 命令行恢复参数
- `--list-sessions` - 列出所有会话

### 2. UI组件集成

```typescript
// 在React组件中使用
const { loadCheckpoint, saveCheckpoint } = useLogger();
const { listSessions, resolveSession } = useSessionUtils();
```

### 3. 工具系统集成

记忆工具与历史记录系统紧密集成：
```typescript
// 记忆保存触发历史记录更新
memoryTool.on('save', (fact) => {
  chatRecordingService.recordMessage({
    type: 'info',
    content: `记忆已保存: "${fact}"`
  });
});
```

## 错误处理

### 1. 文件系统错误

```typescript
try {
  await fs.writeFile(filePath, content);
} catch (error) {
  if (error.code === 'ENOENT') {
    await fs.mkdir(path.dirname(filePath), { recursive: true });
    await fs.writeFile(filePath, content);
  } else {
    throw error;
  }
}
```

### 2. JSON解析错误

```typescript
try {
  const content = JSON.parse(fileContent);
  return content;
} catch {
  // 处理损坏的JSON文件
  await backupCorruptedFile(filePath);
  return createEmptyContent();
}
```

### 3. 兼容性处理

支持旧版本格式的向后兼容：
```typescript
// 处理旧的检查点格式
if (Array.isArray(parsedContent)) {
  return { history: parsedContent as Content[] }; // 转换为新格式
}
```

## 总结

Gemini CLI的历史记录功能是一个多层次、高可靠性的系统，通过以下关键特性实现：

1. **分层架构**：存储层、服务层、工具层清晰分离
2. **数据完整性**：严格的验证和错误处理机制
3. **性能优化**：延迟加载、缓存和增量更新
4. **安全性**：文件名编码和数据验证
5. **可扩展性**：模块化设计便于功能扩展
6. **用户友好**：丰富的会话管理和搜索功能

这套系统确保用户能够无缝地保存、恢复和搜索对话历史，同时保持高性能和数据安全。