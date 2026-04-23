# Go 环境配置教程

本文档记录 CubeFS 项目的 Go 开发环境配置过程。

## 1. Go 安装

### Windows 安装
1. 下载 Go 安装包：https://golang.google.cn/dl/
2. 运行安装程序，默认安装到 `C:\Program Files\Go`
3. 验证安装：
```bash
go version
# 输出示例: go version go1.21.0 windows/amd64
```

## 2. 环境变量配置

### 关键环境变量

| 变量 | 说明 | 推荐值 |
|------|------|--------|
| GOPATH | Go 工作目录，存放依赖包 | `D:\sylar\go\gopath` (放 D 盘避免 C 盘空间不足) |
| GOMODCACHE | 模块缓存目录 | 默认为 `$GOPATH/pkg/mod` |
| GOPROXY | 模块代理 | `https://goproxy.cn,direct` (国内镜像) |

### 配置步骤

**方法一：命令行设置（临时）**
```bash
go env -w GOPATH=D:\sylar\go\gopath
go env -w GOPROXY=https://goproxy.cn,direct
```

**方法二：系统环境变量设置（永久）**
1. 右键"此电脑" → 属性 → 高级系统设置 → 环境变量
2. 在"系统变量"中新建：
   - 变量名：`GOPATH`
   - 变量值：`D:\sylar\go\gopath`
3. 同样添加 `GOPROXY=https://goproxy.cn,direct`
4. 重启终端或 VSCode 生效

### 验证配置
```bash
go env GOPATH GOMODCACHE GOPROXY
# 输出示例:
# D:\sylar\go\gopath
# D:\sylar\go\gopath\pkg\mod
# https://goproxy.cn,direct
```

## 3. 下载项目依赖

进入 CubeFS 项目目录，执行：
```bash
cd d:\sylar\JD\cubefs
go mod tidy
```

依赖会下载到 `GOPATH/pkg/mod` 目录（约 500MB+）。

## 4. Windows 编译限制

### 问题
CubeFS 是为 **Linux 服务器环境**设计的分布式文件系统，在 Windows 上无法完整编译：

```bash
go build ./...
# 报错: build constraints exclude all Go files in .../tcmalloc
```

### 原因
- 依赖 `tcmalloc`（Linux 内存分配器）
- 依赖 `cgo` 和 Linux 系统调用
- FUSE 客户端仅支持 Linux

### Windows 上可以做的事情
- 阅读和理解代码
- 编辑代码
- 编译部分不依赖 Linux 的包：
  ```bash
  go build ./util/...
  go build ./sdk/...
  ```
- 运行部分单元测试

## 5. 完整编译方案

### 方案一：WSL2（推荐）

在 Windows 上安装 WSL2 + Ubuntu：

```powershell
# PowerShell 管理员权限
wsl --install -d Ubuntu
```

WSL2 中安装 Go：
```bash
# Ubuntu 中执行
wget https://golang.google.cn/dl/go1.21.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### 方案二：Docker 编译

使用 CubeFS 官方编译镜像：
```bash
docker run -v /path/to/cubefs:/cubefs -it cubefs/cbfs-build:latest
cd /cubefs
make
```

### 方案三：远程 Linux 服务器

使用 VSCode Remote SSH 连接远程 Linux 服务器进行开发：
1. 安装 VSCode 插件：Remote - SSH
2. 连接远程服务器
3. 在远程环境中编译运行

## 6. 常见问题

### Q1: C 盘空间不足
**原因**：GOPATH 默认在 C 盘用户目录下  
**解决**：将 GOPATH 设置到其他盘符，如 `D:\sylar\go\gopath`

### Q2: 依赖下载慢或超时
**原因**：默认从 proxy.golang.org 下载，国内访问慢  
**解决**：设置国内镜像 `go env -w GOPROXY=https://goproxy.cn,direct`

### Q3: 编译报 tcmalloc 错误
**原因**：tcmalloc 是 Linux 专用库  
**解决**：使用 WSL2、Docker 或 Linux 服务器编译

## 7. 推荐 VSCode 插件

- **Go** (golang.go) — Go 语言官方插件，提供代码补全、跳转、调试等功能
- **Go Test Explorer** — 可视化运行测试
- **GitLens** — Git 增强

安装 Go 插件后，按 `Ctrl+Shift+P` → "Go: Install/Update Tools" 安装所有 Go 工具。

---

*最后更新：2026-04-12*
