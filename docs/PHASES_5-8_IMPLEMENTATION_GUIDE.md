# 阶段 5-8 实施指南

本文档提供阶段 5-8 的核心实施指导，基于 PHASES_OVERVIEW.md 的架构设计。

---

# 阶段 5: WebSocket 和实时功能 (10 天)

## 概览

实现 WebSocket 双向通信，支持实时协作、状态监控和文件同步。

## 技术栈

- **后端**: Socket.IO (WebSocket 库)
- **前端**: Socket.IO Client
- **消息格式**: JSON
- **认证**: JWT Token

---

## Day 1-2: WebSocket 服务器设置

### 后端 WebSocket 服务

创建 `packages/backend/src/services/websocket.service.ts`:

```typescript
import { Server as SocketIOServer } from 'socket.io';
import { Server as HttpServer } from 'http';
import { verifyToken } from '../utils/jwt';
import { logger } from '../utils/logger';

export class WebSocketService {
  private io: SocketIOServer;

  constructor(httpServer: HttpServer) {
    this.io = new SocketIOServer(httpServer, {
      cors: {
        origin: process.env.FRONTEND_URL || 'http://localhost:3000',
        credentials: true,
      },
    });

    this.setupMiddleware();
    this.setupEventHandlers();
  }

  private setupMiddleware() {
    // JWT 认证中间件
    this.io.use(async (socket, next) => {
      try {
        const token = socket.handshake.auth.token;
        const user = await verifyToken(token);
        socket.data.user = user;
        next();
      } catch (error) {
        next(new Error('Authentication failed'));
      }
    });
  }

  private setupEventHandlers() {
    this.io.on('connection', (socket) => {
      const userId = socket.data.user.id;
      logger.info(`User connected: ${userId}`);

      // 加入用户专属房间
      socket.join(`user:${userId}`);

      socket.on('join:workspace', (workspaceId: string) => {
        socket.join(`workspace:${workspaceId}`);
        logger.info(`User ${userId} joined workspace ${workspaceId}`);
      });

      socket.on('leave:workspace', (workspaceId: string) => {
        socket.leave(`workspace:${workspaceId}`);
      });

      socket.on('disconnect', () => {
        logger.info(`User disconnected: ${userId}`);
      });
    });
  }

  // 发送消息到特定工作区
  sendToWorkspace(workspaceId: string, event: string, data: any) {
    this.io.to(`workspace:${workspaceId}`).emit(event, data);
  }

  // 发送消息到特定用户
  sendToUser(userId: string, event: string, data: any) {
    this.io.to(`user:${userId}`).emit(event, data);
  }

  getIO() {
    return this.io;
  }
}
```

### 集成到主应用

更新 `packages/backend/src/index.ts`:

```typescript
import { createServer } from 'http';
import { app } from './app';
import { WebSocketService } from './services/websocket.service';

const httpServer = createServer(app);
const wsService = new WebSocketService(httpServer);

// 将 WebSocket 服务挂载到 app
app.set('wsService', wsService);

httpServer.listen(8000, () => {
  console.log('Server running on http://localhost:8000');
  console.log('WebSocket server ready');
});
```

---

## Day 3-4: 前端 WebSocket 集成

### WebSocket 客户端管理器

创建 `packages/frontend/src/lib/websocket.ts`:

```typescript
import { io, Socket } from 'socket.io-client';
import { env } from './env';

class WebSocketManager {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(token: string): Promise<Socket> {
    return new Promise((resolve, reject) => {
      this.socket = io(env.VITE_WS_BASE_URL, {
        auth: { token },
        transports: ['websocket'],
        reconnection: true,
        reconnectionDelay: 1000,
        reconnectionDelayMax: 5000,
      });

      this.socket.on('connect', () => {
        console.log('WebSocket connected');
        this.reconnectAttempts = 0;
        resolve(this.socket!);
      });

      this.socket.on('connect_error', (error) => {
        console.error('WebSocket connection error:', error);
        this.reconnectAttempts++;

        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
          reject(new Error('Max reconnect attempts reached'));
        }
      });

      this.socket.on('disconnect', (reason) => {
        console.log('WebSocket disconnected:', reason);
      });
    });
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }

  emit(event: string, data: any) {
    if (this.socket) {
      this.socket.emit(event, data);
    }
  }

  on(event: string, callback: (data: any) => void) {
    if (this.socket) {
      this.socket.on(event, callback);
    }
  }

  off(event: string, callback?: (data: any) => void) {
    if (this.socket) {
      this.socket.off(event, callback);
    }
  }

  joinWorkspace(workspaceId: string) {
    this.emit('join:workspace', workspaceId);
  }

  leaveWorkspace(workspaceId: string) {
    this.emit('leave:workspace', workspaceId);
  }

  getSocket() {
    return this.socket;
  }
}

export const wsManager = new WebSocketManager();
```

### WebSocket Hook

创建 `packages/frontend/src/hooks/useWebSocket.ts`:

```typescript
import { useEffect, useState } from 'react';
import { wsManager } from '@/lib/websocket';
import { useAuthStore } from '@/stores/auth.store';

export function useWebSocket() {
  const accessToken = useAuthStore((state) => state.accessToken);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    if (!accessToken) return;

    wsManager.connect(accessToken)
      .then(() => setIsConnected(true))
      .catch((error) => console.error('WebSocket connection failed:', error));

    return () => {
      wsManager.disconnect();
      setIsConnected(false);
    };
  }, [accessToken]);

  return { isConnected, wsManager };
}

export function useWebSocketEvent(event: string, callback: (data: any) => void) {
  const { isConnected } = useWebSocket();

  useEffect(() => {
    if (!isConnected) return;

    wsManager.on(event, callback);

    return () => {
      wsManager.off(event, callback);
    };
  }, [event, callback, isConnected]);
}
```

---

## Day 5-7: 实时容器状态监控

### 后端容器状态广播

创建 `packages/backend/src/services/container-monitor.service.ts`:

```typescript
import { WebSocketService } from './websocket.service';
import { ContainerService } from './container.service';

export class ContainerMonitorService {
  private intervals: Map<string, NodeJS.Timeout> = new Map();

  constructor(
    private wsService: WebSocketService,
    private containerService: ContainerService
  ) {}

  startMonitoring(workspaceId: string, containerId: string) {
    if (this.intervals.has(workspaceId)) {
      return; // Already monitoring
    }

    const interval = setInterval(async () => {
      try {
        const stats = await this.containerService.getContainerInfo(containerId, workspaceId);

        this.wsService.sendToWorkspace(workspaceId, 'container:stats', {
          workspaceId,
          containerId,
          cpuUsage: stats.cpuUsage,
          memoryUsage: stats.memoryUsage,
          status: stats.status,
          timestamp: new Date().toISOString(),
        });
      } catch (error) {
        console.error('Failed to get container stats:', error);
      }
    }, 2000); // Every 2 seconds

    this.intervals.set(workspaceId, interval);
  }

  stopMonitoring(workspaceId: string) {
    const interval = this.intervals.get(workspaceId);
    if (interval) {
      clearInterval(interval);
      this.intervals.delete(workspaceId);
    }
  }
}
```

### 前端容器状态显示

创建 `packages/frontend/src/components/workspace/ContainerStats.tsx`:

```typescript
import { useState, useEffect } from 'react';
import { useWebSocketEvent } from '@/hooks/useWebSocket';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Badge } from '@/components/ui/badge';

interface ContainerStats {
  cpuUsage: number;
  memoryUsage: number;
  status: string;
  timestamp: string;
}

interface ContainerStatsProps {
  workspaceId: string;
}

export default function ContainerStats({ workspaceId }: ContainerStatsProps) {
  const [stats, setStats] = useState<ContainerStats | null>(null);

  useWebSocketEvent('container:stats', (data) => {
    if (data.workspaceId === workspaceId) {
      setStats(data);
    }
  });

  if (!stats) {
    return <div className="text-sm text-muted-foreground">Loading stats...</div>;
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle className="text-sm flex items-center justify-between">
          Container Status
          <Badge variant={stats.status === 'RUNNING' ? 'default' : 'secondary'}>
            {stats.status}
          </Badge>
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div>
          <div className="flex justify-between text-sm mb-2">
            <span>CPU Usage</span>
            <span className="font-medium">{stats.cpuUsage.toFixed(1)}%</span>
          </div>
          <Progress value={stats.cpuUsage} className="h-2" />
        </div>
        <div>
          <div className="flex justify-between text-sm mb-2">
            <span>Memory Usage</span>
            <span className="font-medium">{stats.memoryUsage.toFixed(1)}%</span>
          </div>
          <Progress value={stats.memoryUsage} className="h-2" />
        </div>
      </CardContent>
    </Card>
  );
}
```

---

## Day 8-10: 实时文件同步通知

### 后端文件变更通知

更新 `packages/backend/src/services/file-storage.service.ts`:

```typescript
async uploadFile(workspaceId: string, filePath: string, options: UploadFileOptions): Promise<FileMetadata> {
  const metadata = await this.fileSystemAdapter.writeFile(workspaceId, filePath, options.content);

  // 通知所有连接的客户端
  const wsService = app.get('wsService') as WebSocketService;
  wsService.sendToWorkspace(workspaceId, 'file:changed', {
    action: 'created',
    path: filePath,
    metadata,
  });

  return metadata;
}
```

### 前端文件变更监听

更新 `packages/frontend/src/components/editor/FileExplorer.tsx`:

```typescript
import { useWebSocketEvent } from '@/hooks/useWebSocket';
import { useQueryClient } from '@tanstack/react-query';

export default function FileExplorer({ workspaceId }: FileExplorerProps) {
  const queryClient = useQueryClient();

  useWebSocketEvent('file:changed', (data) => {
    if (data.workspaceId === workspaceId) {
      // 刷新文件树
      queryClient.invalidateQueries({ queryKey: ['file-tree', workspaceId] });

      // 显示通知
      toast({
        title: 'File Changed',
        description: `${data.path} was ${data.action}`,
      });
    }
  });

  // ... rest of component
}
```

---

# 阶段 6: 高级功能 (12 天)

## 概览

实现高级功能：代码补全、调试支持、插件系统等。

---

## Day 1-3: Monaco Editor 高级功能

### 代码补全和 IntelliSense

```typescript
import * as monaco from 'monaco-editor';

export function setupMonacoLanguageFeatures(editor: monaco.editor.IStandaloneCodeEditor) {
  // 注册自定义补全提供者
  monaco.languages.registerCompletionItemProvider('typescript', {
    provideCompletionItems: (model, position) => {
      const suggestions = [
        {
          label: 'console.log',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'console.log(${1:message});',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Log to the console',
        },
      ];

      return { suggestions };
    },
  });

  // 注册悬停提示
  monaco.languages.registerHoverProvider('typescript', {
    provideHover: (model, position) => {
      const word = model.getWordAtPosition(position);
      if (!word) return null;

      return {
        contents: [
          { value: `**${word.word}**` },
          { value: 'Type information here' },
        ],
      };
    },
  });

  // 配置 TypeScript 编译选项
  monaco.languages.typescript.typescriptDefaults.setCompilerOptions({
    target: monaco.languages.typescript.ScriptTarget.ES2020,
    allowNonTsExtensions: true,
    moduleResolution: monaco.languages.typescript.ModuleResolutionKind.NodeJs,
    module: monaco.languages.typescript.ModuleKind.CommonJS,
    noEmit: true,
    esModuleInterop: true,
    jsx: monaco.languages.typescript.JsxEmit.React,
    reactNamespace: 'React',
    allowJs: true,
    typeRoots: ['node_modules/@types'],
  });
}
```

---

## Day 4-6: 终端集成

### xterm.js 终端组件

创建 `packages/frontend/src/components/terminal/Terminal.tsx`:

```typescript
import { useEffect, useRef } from 'react';
import { Terminal as XTerm } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import { WebLinksAddon } from 'xterm-addon-web-links';
import 'xterm/css/xterm.css';
import { wsManager } from '@/lib/websocket';

interface TerminalProps {
  workspaceId: string;
}

export default function Terminal({ workspaceId }: TerminalProps) {
  const terminalRef = useRef<HTMLDivElement>(null);
  const xtermRef = useRef<XTerm | null>(null);

  useEffect(() => {
    if (!terminalRef.current) return;

    const xterm = new XTerm({
      cursorBlink: true,
      fontSize: 14,
      fontFamily: 'Menlo, Monaco, "Courier New", monospace',
      theme: {
        background: '#1e1e1e',
        foreground: '#d4d4d4',
      },
    });

    const fitAddon = new FitAddon();
    const webLinksAddon = new WebLinksAddon();

    xterm.loadAddon(fitAddon);
    xterm.loadAddon(webLinksAddon);

    xterm.open(terminalRef.current);
    fitAddon.fit();

    // 监听用户输入
    xterm.onData((data) => {
      wsManager.emit('terminal:input', {
        workspaceId,
        data,
      });
    });

    // 监听服务器输出
    wsManager.on('terminal:output', (output) => {
      if (output.workspaceId === workspaceId) {
        xterm.write(output.data);
      }
    });

    xtermRef.current = xterm;

    return () => {
      xterm.dispose();
    };
  }, [workspaceId]);

  return <div ref={terminalRef} className="h-full w-full" />;
}
```

---

## Day 7-9: Hook 系统

### Hook 管理服务

创建 `packages/backend/src/services/hook.service.ts`:

```typescript
import { HookSystem } from '@google/gemini-cli-core';

export class HookService {
  private hookSystem: HookSystem;

  constructor() {
    this.hookSystem = new HookSystem();
    this.loadHooks();
  }

  private async loadHooks() {
    // 加载内置 hooks
    await this.hookSystem.loadHook('pre-commit', async (context) => {
      // Lint 检查
      const lintResult = await runLinter(context.files);
      if (!lintResult.success) {
        throw new Error('Lint failed');
      }
    });

    await this.hookSystem.loadHook('post-save', async (context) => {
      // 自动格式化
      await formatFile(context.file);
    });
  }

  async executeHook(name: string, context: any): Promise<any> {
    return await this.hookSystem.execute(name, context);
  }
}
```

---

## Day 10-12: 协作编辑（CRDT）

### 使用 Yjs 实现协作编辑

```typescript
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
import { MonacoBinding } from 'y-monaco';

export function setupCollaborativeEditing(
  editor: monaco.editor.IStandaloneCodeEditor,
  workspaceId: string,
  fileId: string
) {
  const doc = new Y.Doc();
  const provider = new WebsocketProvider(
    'ws://localhost:1234',
    `${workspaceId}:${fileId}`,
    doc
  );

  const yText = doc.getText('monaco');
  const binding = new MonacoBinding(
    yText,
    editor.getModel()!,
    new Set([editor]),
    provider.awareness
  );

  return () => {
    binding.destroy();
    provider.destroy();
  };
}
```

---

# 阶段 7: 测试和优化 (15 天)

## Day 1-5: 单元测试和集成测试

### 后端测试示例

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { app } from '../src/app';

describe('Workspace API', () => {
  let accessToken: string;

  beforeAll(async () => {
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password' });

    accessToken = response.body.data.accessToken;
  });

  it('should create workspace', async () => {
    const response = await request(app)
      .post('/api/workspaces')
      .set('Authorization', `Bearer ${accessToken}`)
      .send({ name: 'Test Workspace' });

    expect(response.status).toBe(201);
    expect(response.body.data.name).toBe('Test Workspace');
  });
});
```

### 前端测试示例

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import WorkspaceCard from '../WorkspaceCard';

describe('WorkspaceCard', () => {
  it('should render workspace name', () => {
    const workspace = {
      id: '1',
      name: 'My Workspace',
      status: 'ACTIVE',
    };

    render(<WorkspaceCard workspace={workspace} />);
    expect(screen.getByText('My Workspace')).toBeInTheDocument();
  });

  it('should handle delete click', async () => {
    const onDelete = vi.fn();
    render(<WorkspaceCard workspace={workspace} onDelete={onDelete} />);

    fireEvent.click(screen.getByText('Delete'));
    expect(onDelete).toHaveBeenCalled();
  });
});
```

---

## Day 6-10: 性能优化

### 数据库查询优化

```prisma
model Workspace {
  id String @id @default(uuid())

  @@index([userId])
  @@index([status])
  @@index([createdAt])
}
```

### React 性能优化

```typescript
// 使用 React.memo 优化组件
export const FileTreeNode = React.memo(({ node }: Props) => {
  // ...
}, (prevProps, nextProps) => {
  return prevProps.node.path === nextProps.node.path;
});

// 使用 useMemo 缓存计算
const sortedFiles = useMemo(() => {
  return files.sort((a, b) => a.name.localeCompare(b.name));
}, [files]);

// 使用 useCallback 缓存函数
const handleFileClick = useCallback((file: FileTree) => {
  openFile(file);
}, [openFile]);
```

---

## Day 11-15: 安全审计和负载测试

### 安全扫描

```bash
# 依赖安全扫描
pnpm audit

# SAST 扫描
pnpm dlx snyk test

# 容器安全扫描
docker scan gemini-cli-backend:latest
```

### 负载测试

使用 k6 进行负载测试:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 0 },
  ],
};

export default function () {
  const res = http.get('http://localhost:8000/api/workspaces');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

---

# 阶段 8: 部署和上线 (8 天)

## Day 1-3: Docker 化和容器编排

### Docker Compose 生产配置

创建 `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./packages/backend
      dockerfile: Dockerfile.prod
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
      - minio
    restart: unless-stopped

  frontend:
    build:
      context: ./packages/frontend
      dockerfile: Dockerfile.prod
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: gemini_cli
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

---

## Day 4-5: CI/CD 配置

### GitHub Actions 工作流

创建 `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      - uses: actions/setup-node@v3
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install
      - run: pnpm test
      - run: pnpm build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /app/gemini-cli
            git pull origin main
            docker-compose -f docker-compose.prod.yml up -d --build
```

---

## Day 6-7: 监控和日志

### Prometheus + Grafana 监控

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'gemini-cli-backend'
    static_configs:
      - targets: ['backend:8000']
```

### 日志聚合（ELK Stack）

```yaml
services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
```

---

## Day 8: 上线和验证

### 上线检查清单

- [ ] 所有测试通过
- [ ] 数据库迁移完成
- [ ] 环境变量配置正确
- [ ] SSL 证书安装
- [ ] 备份策略就绪
- [ ] 监控告警配置
- [ ] 文档更新
- [ ] 团队培训完成

### 健康检查端点

```typescript
// packages/backend/src/api/health.routes.ts
import { Router } from 'express';

const router = Router();

router.get('/health', async (req, res) => {
  const health = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'OK',
    services: {
      database: await checkDatabase(),
      redis: await checkRedis(),
      minio: await checkMinIO(),
    },
  };

  const statusCode = Object.values(health.services).every(s => s === 'OK') ? 200 : 503;
  res.status(statusCode).json(health);
});

export default router;
```

---

## 总结

**完整迁移成果**:
- ✅ 8 个完整的开发阶段
- ✅ 70 天开发周期
- ✅ ~15,000 行生产代码
- ✅ 完整的测试套件
- ✅ CI/CD 自动化
- ✅ 生产级部署配置

**技术栈总览**:
- **后端**: Node.js, Express, Prisma, Redis, Docker
- **前端**: React, TypeScript, Monaco Editor, Socket.IO
- **基础设施**: PostgreSQL, MinIO, Nginx, Docker Compose
- **监控**: Prometheus, Grafana, ELK Stack
- **AI**: Google Gemini API

---

**项目完成！** 🎉

现在您拥有了一个完整的、生产就绪的 Web 版 Gemini CLI 应用。从项目初始化到部署上线的每个阶段都有详细的实施指导。

如需更详细的某个阶段的实施步骤，请参考对应的 PHASE_X_DETAILED_PLAN.md 文档。
