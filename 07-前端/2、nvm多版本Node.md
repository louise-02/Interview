Windows 下用 **nvm-windows** 安装、切换多版本 Node。

---

# 1、下载

https://github.com/coreybutler/nvm-windows/releases

安装后新开终端，用 `nvm version` 验证。

---

# 2、常用命令

```bash
# 查看已安装版本
nvm list
nvm list installed

# 查看可安装版本
nvm list available

# 安装
nvm install 14.21.3
nvm install latest

# 切换当前使用的 Node（需管理员终端时以实际为准）
nvm use 14.21.3

# 卸载
nvm uninstall 14.21.3
```

| 命令 | 说明 |
| ---- | ---- |
| `nvm on` / `nvm off` | 开启 / 关闭 nvm 管理 |
| `nvm proxy [url]` | 设置下载代理 |
| `nvm root [path]` | 各版本 Node 存放目录 |

---

# 3、镜像（国内加速）

```bash
nvm node_mirror https://npmmirror.com/mirrors/node/
nvm npm_mirror https://npmmirror.com/mirrors/npm/
```

> 旧地址 `npm.taobao.org/mirrors/...` 已迁移到 **npmmirror.com**。

---

# 4、注意

| 点 | 说明 |
| -- | ---- |
| 换版本后 | 建议删项目 `node_modules`，再 `npm install` |
| 全局包 | 每个 Node 版本独立；`nvm use` 切换后全局模块需重装 |
| install 报错 | node-sass、python 等见 [1、npm与VueCLI常用 §6 常见报错](./1、npm与VueCLI常用.md) |
