
---
## Prisma 二进制引擎丢失

**根因**：Prisma 会按宿主机平台生成原生 `.node` / `.dylib.node` 引擎文件。打包后运行平台与构建平台不一致，或 electron-builder 将引擎文件排在 ASAR 之外路径错位，导致运行时找不到二进制。

#### Step 1 — `schema.prisma` 声明目标平台

```prisma
// prisma
generator client {
  provider      = "prisma-client-js"
  engineType    = "library"          // 用动态库而非独立进程，更适合 Electron
  binaryTargets = [
    "native",                        // 构建机本地（开发用）
    "darwin",                        // macOS Intel
    "darwin-arm64",                  // macOS Apple Silicon
    "windows",                       // Windows x64
    "debian-openssl-3.0.x"           // Linux（常见 CI/CD 容器）
  ]
}
```

运行 `npx prisma generate` 后，`.prisma/client/` 下会同时存在多平台的 `.node` 文件。

---

#### Step 2 — `electron-builder` 配置：把引擎文件解包到 `asarUnpack`

```json
// package.json → build 段
{
  "build": {
    "asar": true,
    "asarUnpack": [
      "node_modules/.prisma/client/**",
      "node_modules/@prisma/engines/**"
    ],
    "extraResources": [
      {
        "from": "node_modules/.prisma/client",
        "to": "prisma/client"
      }
    ]
  }
}
```

> ASAR 是只读虚拟文件系统，原生 `.node` 模块必须在真实文件路径下才能被 `dlopen` 加载，所以必须 unpack。

---

#### Step 3 — 运行时动态修正引擎路径

```typescript
// main/prisma.ts
import path from 'path'
import { app } from 'electron'
import { PrismaClient } from '@prisma/client'

function getEngineDir(): string {
  // 打包后：resources/prisma/client
  // 开发时：node_modules/.prisma/client
  return app.isPackaged
    ? path.join(process.resourcesPath, 'prisma/client')
    : path.join(__dirname, '../../node_modules/.prisma/client')
}

// 在实例化前设置，Prisma 内部会读这个变量
process.env.PRISMA_QUERY_ENGINE_LIBRARY = path.join(
  getEngineDir(),
  // 文件名随平台自动区分
  `libquery_engine-${getPlatformTarget()}.dylib.node`
)

function getPlatformTarget(): string {
  if (process.platform === 'darwin') {
    return process.arch === 'arm64' ? 'darwin-arm64' : 'darwin'
  }
  if (process.platform === 'win32') return 'windows'
  return 'debian-openssl-3.0.x'
}

export const prisma = new PrismaClient({
  datasources: {
    db: { url: `file:${path.join(app.getPath('userData'), 'app.db')}` }
  }
})
```

---

## 符号链接悬空（Dangling Symlinks）

**根因**：pnpm / yarn workspaces 大量使用 symlink 来共享包。electron-builder 拷贝文件时默认不跟随符号链接，导致 ASAR 内的链接指向不存在的目标。

#### 对特定包彻底关闭 ASAR

```json
{
  "build": {
    "asar": true,
    "asarUnpack": [
      // 明确列出含 symlink 的包，让它们以真实目录存在于磁盘
      "node_modules/better-sqlite3/**",
      "node_modules/.prisma/**"
    ]
  }
}
```

---

## 子进程环境变量隔离

**根因**：Electron 主进程通过 `spawn` 启动 Next.js standalone 服务时，子进程会继承主进程 `process.env`，但在生产打包环境下，`DATABASE_URL`、`PRISMA_` 等变量从未被写入，子进程拿到的是空值。

#### 完整封装：`startNextServer`

```typescript
// main/server-manager.ts
import { spawn, ChildProcess } from 'child_process'
import path from 'path'
import { app } from 'electron'

let nextServer: ChildProcess | null = null

export function startNextServer(port = 3456): Promise<void> {
  return new Promise((resolve, reject) => {
    const serverEntry = app.isPackaged
      ? path.join(process.resourcesPath, 'app/.next/standalone/server.js')
      : path.join(__dirname, '../../.next/standalone/server.js')

    // ① 构造子进程专属环境变量，不依赖继承
    const childEnv: NodeJS.ProcessEnv = {
      ...process.env,               // 保留系统路径等基础变量

      // ② 数据库连接：开发走 .env，生产指向 userData（可写目录）
      DATABASE_URL: app.isPackaged
        ? `file:${path.join(app.getPath('userData'), 'app.db')}`
        : (process.env.DATABASE_URL ?? 'file:./dev.db'),

      // ③ Prisma 引擎路径（对应问题一的解法）
      PRISMA_QUERY_ENGINE_LIBRARY: getPrismaEnginePath(),

      // ④ Next.js 运行时必要变量
      NODE_ENV: 'production',
      PORT: String(port),
      HOSTNAME: '127.0.0.1',

      // ⑤ 其他业务密钥（打包时从安全存储读取，而非硬编码）
      NEXTAUTH_SECRET: getSecretFromKeychain(),
      NEXTAUTH_URL: `http://127.0.0.1:${port}`,
    }

    nextServer = spawn(process.execPath, [serverEntry], {
      env: childEnv,
      stdio: ['ignore', 'pipe', 'pipe'],
    })

    // ⑥ 等待服务就绪后再 resolve
    nextServer.stdout?.on('data', (chunk: Buffer) => {
      const msg = chunk.toString()
      console.log('[next]', msg.trim())
      if (msg.includes('Ready') || msg.includes(`${port}`)) resolve()
    })

    nextServer.stderr?.on('data', (chunk: Buffer) => {
      console.error('[next:err]', chunk.toString().trim())
    })

    nextServer.on('error', reject)
    nextServer.on('exit', (code) => {
      console.warn(`[next] 子进程退出，code=${code}`)
      nextServer = null
    })
  })
}

export function stopNextServer() {
  nextServer?.kill('SIGTERM')
  nextServer = null
}

// 从系统 Keychain / safeStorage 读取，避免写死在代码里
function getSecretFromKeychain(): string {
  const { safeStorage } = require('electron')
  // 实际项目里改成真实的 keychain 读取逻辑
  return safeStorage.isEncryptionAvailable()
    ? safeStorage.decryptString(Buffer.from(process.env._ENCRYPTED_SECRET ?? '', 'base64'))
    : process.env.NEXTAUTH_SECRET ?? 'dev-secret'
}

function getPrismaEnginePath(): string {
  const dir = app.isPackaged
    ? path.join(process.resourcesPath, 'prisma/client')
    : path.join(__dirname, '../../node_modules/.prisma/client')

  const suffix = process.platform === 'darwin'
    ? (process.arch === 'arm64' ? 'darwin-arm64' : 'darwin')
    : process.platform === 'win32' ? 'windows' : 'debian-openssl-3.0.x'

  return path.join(dir, `libquery_engine-${suffix}.dylib.node`)
}
```

---

#### 三者协作关系总结

```
schema.prisma          → 声明 binaryTargets，prisma generate 生成多平台引擎文件
        ↓
electron-builder       → asarUnpack 把 .node 文件解包到真实磁盘路径
afterPack 钩子         → 展开 symlink，确保所有依赖路径都是真实文件
        ↓
startNextServer()      → 显式构造 childEnv，把 DATABASE_URL / PRISMA_* / PORT
                         注入子进程，子进程不再依赖"隐式继承"
```

三个问题的核心思路一致：**不信任运行时的隐式推断，每一个关键路径/变量都在构建阶段或启动阶段显式指定。**