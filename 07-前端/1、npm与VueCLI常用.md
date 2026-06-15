Vue CLI 项目本地开发：**package.json scripts、安装依赖、启动、打包**，以及 npm 全局/镜像配置。

---

# 1、package.json scripts

```json
{
  "scripts": {
    "start": "npm run dev",
    "dev": "vue-cli-service serve",
    "build:prod": "vue-cli-service build"
  }
}
```

| 命令 | 说明 |
| ---- | ---- |
| `npm run dev` | 本地开发，热更新 |
| `npm run build:prod` | 生产打包，产物一般在 `dist/` |
| `npm start` | 若配置了 `"start": "npm run dev"`，等价于 dev |

---

# 2、常用命令

```bash
# 安装依赖（根据 package-lock.json / package.json）
npm install

# 启动开发
npm run dev

# 打包
npm run build:prod
# 或（取决于 scripts 是否定义 build）
npm run build -- --mode production
```

---

# 3、npm 环境配置

```bash
# 查看当前配置
npm config list

# 全局包装目录（Windows 示例）
npm config set prefix "d:\devware\node_x64\node_global"

# 缓存目录
npm config set cache "d:\devware\node_x64\node_cache"

# 国内镜像（原淘宝源，现 npmmirror）
npm config set registry https://registry.npmmirror.com
npm config get registry
```

设置 `prefix` 后，把 `%prefix%\node_global` 加到系统 **PATH**，全局命令（如 `vue`、`cnpm`）才能直接用。

---

# 4、cnpm（可选）

```bash
npm install -g cnpm --registry=https://registry.npmmirror.com
```

| 参数 | 说明 |
| ---- | ---- |
| `-g` | 全局安装 |
| `--registry` | 指定安装 cnpm 时使用的源 |

项目里仍可用 `cnpm install` 代替 `npm install`，锁文件与团队保持一致即可（团队统一用一种）。

---

# 5、npm i 与 npm install

**现在二者是同一命令的别名**（`npm i` = `npm install`）：

- 装依赖、写 `package.json` / `package-lock.json` 行为一致
- 卸载都是：`npm uninstall 包名` 或 `npm un 包名`

不存在「用 i 装的包要用 `npm uninstall i` 才能删」这种说法；若遇到异常，多半是包名或目录搞错了。

---

# 6、常见报错

`npm install` / `npm run build` 时常见；多数和 **原生模块编译**、**node-sass 与 Node 版本** 有关。

## python / node-gyp（windows-build-tools）

编译原生模块（如老版 node-sass）需要 Python 与 C++ 构建工具：

```bash
# 管理员 PowerShell（老方案，Win10 仍有人用）
npm install --global --production windows-build-tools
```

Win11 / 新环境更推荐：**安装 Visual Studio Build Tools**（勾选「使用 C++ 的桌面开发」），并切到与项目匹配的 Node 版本（见 [2、nvm多版本Node](./2、nvm多版本Node.md)）再 `npm install`。

## node-sass 报错（方案 1）

```bash
npm uninstall node-sass
npm install node-sass --save
```

## node-sass 报错（方案 2）

```bash
npm uninstall node-sass
cnpm install node-sass@latest --save
```

## 新项目建议

Vue CLI 5+ / Vite 项目优先用 **`sass`（dart-sass）**，不要用 `node-sass`：

```bash
npm uninstall node-sass
npm install sass -D
```

scss 文件写法不变，构建链更省心。

| 点 | 说明 |
| -- | ---- |
| 版本匹配 | node-sass 与 Node 大版本强相关，装不上先换 Node 再重装依赖 |
| 清缓存 | 仍失败可删 `node_modules`、删 lock 后重装，或 `npm cache clean --force` |

