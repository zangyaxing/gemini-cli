# Gemini CLI 历史记录功能实现详解

## 概述

Gemini CLI 实现了多层次的历史记录功能，包括对话历史记录、命令历史记录、会话检查点以及长期记忆存储。这些功能共同构成了一个完整的会话管理和历史追溯系统。

## 核心组件架构

### 1. 存储层 (Storage Layer)

**位置**: `packages/core/src/config/storage.ts`

存储层是所有历史记录功能的基础，负责定义文件路径和目录结构：

```typescript
export class Storage {
  // 获取项目临时目录，用于存储历史记录
  getProjectTempDir(): string {
    const hash = this.getFilePathHash(this.getProjectRoot());
    const tempDir = Storage.getGlobalTempDir();
    return path.join(tempDir, hash);
  }

  // 获取历史记录文件路径
  getHistoryFilePath(): string {
    return path.join(this.getProjectTempDir(), 'shell_history');
  }

  // 获取检查点存储目录
  getProjectTempCheckpointsDir(): string {
    return path.join(this.getProjectTempDir(), 'checkpoints');
  }
}
```

**存储结构**:
- 全局目录: `~/.gemini/`
- 项目临时目录: `~/.gemini/tmp/<项目hash>/`
- 历史记录文件: `~/.gemini/tmp/<项目hash>/shell_history`
- 检查点目录: `~/.gemini/tmp/<项目hash>/checkpoints/`
- 日志文件: `~/.gemini/tmp/<项目hash>/logs.json`

### 2. 日志记录器 (Logger)

**位置**: `packages/core/src/core/logger.ts`

Logger 是历史记录功能的核心组件，负责管理会话日志和检查点：

```typescript
export class Logger {
  private logs: LogEntry[] = []; // 内存缓存
  private sessionId: string;
  private messageId = 0; // 消息ID计数器

  // 记录消息到日志
  async logMessage(type: MessageSenderType, message: string): Promise<void> {
    const newEntryObject: LogEntry = {
      sessionId: this.sessionId,
      messageId: this.messageId,
      type,
      message,
      timestamp: new Date().toISOString(),
    };
    await this._updateLogFile(newEntryObject);
  }

  // 保存检查点
  async saveCheckpoint(checkpoint: Checkpoint, tag: string): Promise<void> {
    const path = this._checkpointPath(tag);
    await fs.writeFile(path, JSON.stringify(checkpoint, null, 2), 'utf-8');
  }

  // 加载检查点
  async loadCheckpoint(tag: string): Promise<Checkpoint> {
    const path = await this._getCheckpointPath(tag);
    const fileContent = await fs.readFile(path, 'utf-8');
    const parsedContent = JSON.parse(fileContent);
    return { history: parsedContent as Content[] };
  }
}
```

**关键特性**:
- **会话隔离**: 每个会话有唯一的 sessionId
- **消息ID管理**: 自动递增的消息ID确保消息顺序
- **错误恢复**: 自动备份损坏的日志文件
- **检查点编码**: 使用 URL 编码处理特殊字符

### 3. Shell 历史记录 (Shell History)

**位置**: `packages/cli/src/ui/hooks/useShellHistory.ts`

Shell 历史记录管理用户的命令输入历史：

```typescript
export function useShellHistory(projectRoot: string, storage?: Storage): UseShellHistoryReturn {
  const [history, setHistory] = useState<string[]>([]);
  const [historyIndex, setHistoryIndex] = useState(-1);

  // 读取历史文件
  async function readHistoryFile(filePath: string): Promise<string[]> {
    const text = await fs.readFile(filePath, 'utf-8');
    // 处理多行命令（以反斜杠结尾的续行）
    const result: string[] = [];
    let cur = '';
    for (const raw of text.split(/\r?\n/)) {
      const m = cur.match(/(\\+)$/);
      if (m && m[1].length % 2) {
        // 奇数个反斜杠表示续行
        cur = cur.slice(0, -1) + ' ' + line;
      } else {
        if (cur) result.push(cur);
        cur = line;
      }
    }
    return result;
  }

  // 添加命令到历史
  const addCommandToHistory = useCallback((command: string) => {
    const newHistory = [command, ...history.filter((c) => c !== command)]
      .slice(0, MAX_HISTORY_LENGTH);
    setHistory(newHistory);
    writeHistoryFile(historyFilePath, [...newHistory].reverse());
  }, [history, historyFilePath]);
}
```

**功能特点**:
- **多行命令支持**: 正确处理以反斜杠续行的命令
- **历史长度限制**: 最多保存 100 条命令
- **去重处理**: 自动过滤重复命令
- **键盘导航**: 支持上下键浏览历史

### 4. 检查点系统 (Checkpoint System)

**位置**: `packages/cli/src/ui/commands/restoreCommand.ts`

检查点系统允许用户保存和恢复会话状态：

```typescript
async function restoreAction(context: CommandContext, args: string): Promise<void> {
  const checkpointDir = config?.storage.getProjectTempCheckpointsDir();
  const files = await fs.readdir(checkpointDir);
  const jsonFiles = files.filter((file) => file.endsWith('.json'));

  if (!args) {
    // 列出可用检查点
    return {
      type: 'message',
      messageType: 'info',
      content: `Available tool calls to restore:\n\n${fileList}`,
    };
  }

  // 恢复特定检查点
  const filePath = path.join(checkpointDir, selectedFile);
  const toolCallData = JSON.parse(data);

  if (toolCallData.history) {
    loadHistory(toolCallData.history); // 恢复对话历史
  }

  if (toolCallData.clientHistory) {
    await config?.getGeminiClient()?.setHistory(toolCallData.clientHistory);
  }

  if (toolCallData.commitHash) {
    await gitService?.restoreProjectFromSnapshot(toolCallData.commitHash);
  }
}
```

**检查点内容**:
- **对话历史**: 完整的用户与AI对话记录
- **客户端历史**: Gemini客户端的内部状态
- **Git快照**: 项目文件的完整状态快照
- **工具调用**: 待执行的工具调用信息

### 5. 长期记忆存储 (Memory Storage)

**位置**: `packages/core/src/tools/memoryTool.ts`

长期记忆功能允许AI记住跨会话的重要信息：

```typescript
export class MemoryTool extends BaseDeclarativeTool<SaveMemoryParams, ToolResult> {
  static async performAddMemoryEntry(
    text: string,
    memoryFilePath: string,
    fsAdapter: FileSystemAdapter,
  ): Promise<void> {
    let currentContent = '';
    try {
      currentContent = await fsAdapter.readFile(memoryFilePath, 'utf-8');
    } catch (_e) {
      // 文件不存在是正常情况
    }

    const newContent = computeNewContent(currentContent, text);
    await fsAdapter.writeFile(memoryFilePath, newContent, 'utf-8');
  }
}

function computeNewContent(currentContent: string, fact: string): string {
  const newMemoryItem = `- ${fact.trim()}`;
  const headerIndex = currentContent.indexOf(MEMORY_SECTION_HEADER);

  if (headerIndex === -1) {
    // 没有找到记忆部分，添加新部分
    return currentContent + `${MEMORY_SECTION_HEADER}\n${newMemoryItem}\n`;
  } else {
    // 在现有记忆部分添加新条目
    // ... 插入逻辑
  }
}
```

**记忆存储特性**:
- **Markdown格式**: 使用 `GEMINI.md` 文件存储记忆
- **结构化存储**: 在 "## Gemini Added Memories" 部分存储
- **确认机制**: 支持用户确认记忆内容
- **跨会话持久**: 记忆在会话间保持可用

## 历史记录工作流程

### 1. 会话初始化

```typescript
// 用户启动Gemini CLI
const sessionId = generateSessionId();
const logger = new Logger(sessionId, storage);
await logger.initialize();

// 加载Shell历史
const { history } = useShellHistory(projectRoot, storage);
```

### 2. 对话记录

```typescript
// 用户输入消息
await logger.logMessage(MessageSenderType.USER, userMessage);

// AI响应
await logger.logMessage(MessageSenderType.MODEL, aiResponse);

// 工具调用执行
if (toolCall.name === 'save_memory') {
  await memoryTool.execute(params);
}
```

### 3. 检查点创建

```typescript
// 在文件修改操作前自动创建检查点
if (config.checkpointing.enabled) {
  const history = chat.getHistory();
  const authType = config?.getContentGeneratorConfig()?.authType;
  await logger.saveCheckpoint({ history, authType }, tag);
  
  // 创建Git快照
  const commitHash = await gitService.createSnapshot();
}
```

### 4. 会话恢复

```typescript
// 用户执行 /restore 命令
const checkpoint = await logger.loadCheckpoint(tag);
loadHistory(checkpoint.history);
await geminiClient.setHistory(checkpoint.clientHistory);
await gitService.restoreProjectFromSnapshot(checkpoint.commitHash);
```

## 配置选项

历史记录功能通过 `settings.json` 配置：

```json
{
  "general": {
    "checkpointing": {
      "enabled": true,
      "autoSave": true
    },
    "sessionRetention": {
      "enabled": true,
      "maxSessionAge": "30d",
      "maxSessionCount": 100
    }
  },
  "model": {
    "maxSessionTurns": 50
  },
  "ui": {
    "useAlternateBuffer": false
  }
}
```

## 安全性和隐私

### 1. 本地存储
- 所有历史记录都存储在本地文件系统
- 不会上传到云端服务器
- 用户完全控制数据的存储和删除

### 2. 数据隔离
- 不同项目的历史记录完全隔离
- 使用项目hash确保数据不会混淆
- 会话ID确保不同会话的独立性

### 3. 错误处理
- 自动备份损坏的日志文件
- 优雅处理文件读写错误
- 提供数据恢复机制

## 性能优化

### 1. 内存缓存
- 日志记录器在内存中维护日志缓存
- 减少频繁的文件I/O操作
- 批量写入提高性能

### 2. 历史限制
- Shell历史限制为100条
- 会话轮次限制防止内存溢出
- 自动清理过期数据

### 3. 延迟加载
- 按需加载历史记录
- 启动时不会立即加载所有数据
- 分页处理大量历史记录

## 总结

Gemini CLI的历史记录功能是一个多层次、高度集成的系统，通过以下组件协同工作：

1. **Storage层**: 提供统一的存储抽象和路径管理
2. **Logger**: 管理会话日志和检查点的核心组件
3. **Shell History**: 处理命令输入历史的用户界面
4. **Checkpoint System**: 提供会话状态的保存和恢复
5. **Memory Tool**: 实现跨会话的长期记忆存储

这个系统设计考虑了性能、安全性和用户体验，为用户提供了完整的历史记录管理能力。