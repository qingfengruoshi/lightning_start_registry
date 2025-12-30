# 🧩 Lightning Start 插件开发指南

欢迎来到 Lightning Start 插件开发世界！通过编写插件，你可以无限扩展启动器的能力。

## 1. 插件结构

一个标准的插件是一个包含 `package.json` 的文件夹（或 `.zip` 压缩包）。

```
my-plugin/
├── package.json       # [必须] 插件配置文件
├── icon.png           # [可选] 插件图标
├── index.js / main.exe # [可选] 插件入口（如果是脚本或可执行文件）
└── README.md          # [推荐] 说明文档
```

## 2. package.json 配置

`package.json` 是插件的核心，定义了插件的元数据和行为。

```json
{
  "name": "plugin-demo",          // [必需] 插件ID，建议使用英文，且全局唯一
  "version": "1.0.0",             // [必需] 版本号
  "description": "这是一个示例插件", // [必需] 简短描述
  "author": "Your Name",          // [推荐] 作者
  "main": "index.js",             // [可选] 入口文件
  "icon": "icon.png",             // [可选] 图标文件路径
  "features": [                   // [核心] 功能定义
    {
      "code": "demo",             // 触发关键词
      "explain": "示例功能",       // 搜索时的说明
      "cmds": ["demo", "hello"]   // 触发此插件的关键词列表
    }
  ]
}
```

## 3. 插件类型

目前支持以下类型的插件集成：

### A. Node.js 模块 (推荐)
这是目前功能最强大的开发方式。插件运行在 Electron 的 **主进程 (Main Process)** 中，拥有完整的 Node.js 能力 (`fs`, `child_process`, `net` 等)。

**入口文件 (index.js) 规范**：
你需要导出一个遵循 `ExternalPluginModule` 接口的对象。

```javascript
const { shell } = require('electron');

module.exports = {
    // [必需] 搜索处理函数
    // query: 用户输入的关键词（已去除触发词）
    search: async (query) => {
        // 返回一个 Promise，resolve 结果数组
        return [
            {
                id: 'demo-1',
                title: 'Hello ' + query,
                subtitle: '这是一个测试结果',
                icon: 'icon.png', // 相对路径或绝对路径
                data: { text: query } // 传递给 execute 的数据
            }
        ];
    },

    // [可选] 执行处理函数
    // item: 用户选中结果的 data 字段
    // { db }: 注入的上下文对象，包含数据库 API
    execute: async (item, { db }) => {
        // 例如：打开浏览器
        console.log('用户点击了：', item.text);
        
        // 示例：使用数据库存储点击次数
        const count = db.get('clickCount', 0);
        db.set('clickCount', count + 1);
        console.log('累计点击次数：', db.get('clickCount'));
        
        // shell.openExternal('https://google.com/search?q=' + item.text);
    },

    // [可选] 生命周期
    onLoad: ({ db }) => { 
        console.log('插件已加载');
        // 初始化配置
        if (!db.has('firstRun')) {
            db.set('firstRun', false);
            console.log('首次运行插件！');
        }
    },
    onUnload: () => { console.log('插件已卸载'); }
};
```

需要在 `package.json` 中添加 `antigravity` 字段来声明元数据：
```json
{
  "name": "plugin-demo",
  "main": "index.js",
  "antigravity": {
    "title": "示例插件",
    "description": "一个 Node.js 插件示例",
    "icon": "icon.png",
    "triggers": ["demo"]
  }
}
```

### 数据库 API (DB)
每个插件都拥有一个独立的 JSON 数据库，数据持久化存储在用户目录下。
通过 context 参数注入的 `db` 对象使用：

*   `db.get(key, defaultValue)`: 获取数据
*   `db.set(key, value)`: 写入数据
*   `db.has(key)`: 检查键是否存在
*   `db.delete(key)`: 删除数据

```javascript
search: async (query, { db }) => {
   const lastQuery = db.get('lastQuery');
   db.set('lastQuery', query);
   // ...
}
```

### B. 外部应用/脚本型 (Legacy)
(旧版方式，建议使用 Node.js 模块方式调用 `child_process` 自行实现，更加灵活)

## 4. 开发与测试

1.  **创建目录**：在你的电脑上创建一个文件夹，例如 `my-plugin`。
2.  **编写配置**：新建 `package.json` 并填入上述信息。
3.  **安装到本地**：
    *   打开 Lightning Start 设置 -> **插件中心** -> **本地插件**。
    *   点击 **📂 打开插件目录**。
    *   将你的 `my-plugin` 文件夹直接复制进去。
    *   **刷新** 插件列表即可看到你的插件。

## 5. 发布插件

想要分享你的插件？

1.  **打包**：将插件目录压缩为 `.zip` 文件（例如 `plugin-demo.zip`）。
2.  **提交**：
    *   Fork 官方插件仓库 (GitHub: `lightning-start/registry`).
    *   将你的插件信息添加到 `plugins.json`。
    *   提交 Pull Request。

## 6. 示例：快速搜索插件

假设你想做一个快速打开 Google 搜索的插件。

**package.json**:
```json
{
  "name": "google-search",
  "version": "1.0.0",
  "description": "快速 Google 搜索",
  "features": [
    {
      "code": "g",
      "explain": "Google 搜索",
      "cmds": ["g"]
    }
  ]
  // 实际逻辑均由此处定义的 cmds 触发
}
```

## 7. 进阶：如何在一个仓库管理多个插件 (Monorepo)
如果你不想为每个插件都创建一个新仓库，完全可以建立一个“插件大合集”仓库。

**推荐结构**：
```
my-plugins-collection/
├── plugin-a/
│   ├── package.json
│   └── icon.png
├── plugin-b/
│   ├── package.json
│   └── index.js
└── README.md
```

**发布流程**：
1.  分别进入 `plugin-a` 和 `plugin-b` 文件夹。
2.  分别打包成 `plugin-a.zip` 和 `plugin-b.zip`。
3.  在 GitHub Releases 中上传这两个 zip 文件。
4.  在 `registry` 提交时，分别填入各自的下载链接即可。
