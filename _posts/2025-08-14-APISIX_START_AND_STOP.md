---
title: Apache APISIX：启动流程
date: 2025-08-14 14:29:00 +0800
categories: [TEAHICAL, GATEWAY]
tags: [apisix]
---

## 前言

本系列博客旨在通过手撕源码的方式理解 APISIX 的核心运行原理，将涵盖 APISIX 的启动流程、请求的生命周期、插件体系等关键技术点。

如无特殊说明，本文以 APISIX 3.13 版本为例，可通过以下命令获取源码：

```bash
git clone https://github.com/apache/apisix.git --branch=release/3.13
```

作为系列博客的第一篇，我们将聚焦 APISIX 的启动流程，了解它是如何启动的，知道其底层是运行了什么服务。

---

## 构建 APISIX DEV 环境

在解读整体流程前，推荐搭建好 APISIX 的开发环境。

可参考官方文档：[Devcontainer 构建 APISIX 开发环境](https://apisix.apache.org/docs/apisix/build-apisix-dev-environment-devcontainers/)

---

## 启动流程

### 启动命令

在 devcontainer 中可以通过以下方式启动 APISIX 服务：

```bash
make run
# 或者
./bin/apisix start
```

APISIX 默认会监听 9080 端口，可以用 curl 验证：

```bash
curl localhost:9080
```

返回结果

```json
{"error_msg":"404 Route Not Found"}
```

出现这个提示说明 APISIX 已经成功启动。

如果查看 Makefile，可以发现 make run 最终调用的也是：

```bash
ENV_APISIX             ?= $(CURDIR)/bin/apisix

### run : Start the apisix server
.PHONY: run
run: runtime
	@$(call func_echo_status, "$@ -> [ Start ]")
	$(ENV_APISIX) start        // 最终也是 ./bin/apisix start
	@$(call func_echo_success_status, "$@ -> [ Done ]")
```

因此，APISIX 的真正启动入口就是 **bin/apisix** 脚本，它负责后续的初始化流程。

---

### bin/apisix

`bin/apisix` 脚本主要做了以下几件事：

1. 根据安装方式定位 `apisix.lua` 文件
```bash
./apisix/cli/apisix.lua         # 源码安装
/usr/local/share/lua/5.1/...    # Luarocks 安装
/usr/local/apisix/...           # 官方 RPM 或 Docker
```
2. 查找 OpenResty 的可执行路径，并判断版本是否 >= 1.19：
```bash
OR_BIN=$(command -v openresty || exit 1)
OR_EXEC=${OR_BIN:-'/usr/local/openresty-debug/bin/openresty'}
OR_VER=$(openresty -v 2>&1 | awk -F '/' '{print $2}' | awk -F '.' '{print $1 * 100 + $2}')
```
3. 核心启动命令
```bash
exec $LUAJIT_BIN $APISIX_LUA $*
```
其中：
- $LUAJIT_BIN：luajit 二进制文件路径
- $APISIX_LUA：./apisix/cli/apisix.lua
- $*：传入的命令参数（例如 start）

也就是说，bin/apisix 本身只是一个启动入口脚本，最终是通过 luajit 执行 Lua 代码来启动的，即执行 apisix/cli/apisix.lua。

---

### apisix/cli/apisix.lua

我们看下 `apisix.lua` 有哪些内容：

```lua
local pkg_cpath_org = package.cpath
local pkg_path_org = package.path

local _, find_pos_end = string.find(pkg_path_org, ";", -1, true)
if not find_pos_end then
    pkg_path_org = pkg_path_org .. ";"
end

local apisix_home = "/usr/local/apisix"
local pkg_cpath = apisix_home .. "/deps/lib64/lua/5.1/?.so;"
                  .. apisix_home .. "/deps/lib/lua/5.1/?.so;"
local pkg_path_deps = apisix_home .. "/deps/share/lua/5.1/?.lua;"
local pkg_path_env = apisix_home .. "/?.lua;"

-- modify the load path to load our dependencies
package.cpath = pkg_cpath .. pkg_cpath_org
package.path  = pkg_path_deps .. pkg_path_org .. pkg_path_env

-- pass path to construct the final result
local env = require("apisix.cli.env")(apisix_home, pkg_cpath_org, pkg_path_org)
local ops = require("apisix.cli.ops")

ops.execute(env, arg)
```

关键点解析

1. **设置 Lua 依赖路径**

   将 /usr/local/apisix/deps 下的 Lua 文件和 .so 动态库加入到 package.path 与 package.cpath，以确保依赖可加载。

2. **构建运行环境**

   apisix.cli.env 返回启动所需环境信息，其中最核心的是 **openresty_args**：

   ```lua
   local res, err = util.execute_cmd("command -v openresty")
   local openresty_path_abs = util.trim(res)
   
   local openresty_args = openresty_path_abs .. " -p " .. apisix_home .. " -c " .. apisix_home .. "/conf/nginx.conf"
   ```

   这里用到了 `util.execute_cmd`：

   ```lua
   local function execute_cmd(cmd)
       local t, err = popen(cmd)      -- 运行系统命令
       if not t then return nil, err end
   
       local data, err = t:read("*all") -- 读取输出
       t:close()
       return data
   end
   ```

   它的作用就是 **执行 shell 命令并返回结果**。

   最终 openresty_args 形如：

   ```lua
   /usr/bin/openresty -p /usr/local/apisix -c /usr/local/apisix/conf/nginx.conf
   /usr/bin/openresty -p /workspace -c /workspace/conf/nginx.conf
   ...
   ```
   这里目录的差别主要是 apisix_home 变量在起作用：
   - apisix_home 的定位

     apisix.cli.env 中有如下逻辑：

     ```lua
     local script_path = arg[0]
         if script_path:sub(1, 2) == './' then
             apisix_home = util.trim(util.execute_cmd("pwd"))
             if not apisix_home then
                 error("failed to fetch current path")
             end
     ```
     
     - arg[0] 就是 bin/apisix 脚本传给 luajit 的入口参数（当源码安装即 ./apisix/cli/apisix.lua）
     - 当入口是 ./apisix/cli/apisix.lua 这种相对路径时，apisix_home 会被重写为 **当前工作目录**（通过 pwd 获取）。
     - 在 devcontainer 场景下，这个目录就是 /workspace。
   
4. 执行命令分发

   调用 apisix.cli.ops.execute 来处理参数，例如 start、stop、reload 等。

---

### apisix.cli.ops

`ops.execute` 负责处理命令行参数：

```lua
function _M.execute(env, arg)
    local cmd_action = arg[1]  -- arg[1] 也就是传递给 bin/apisix 的第一个参数
    if not cmd_action then
        return help()
    end

    if not action[cmd_action] then
        stderr:write("invalid argument: ", cmd_action, "\n")
        return help()
    end

    action[cmd_action](env, arg[2]) -- 将剩余参数传递给具体的方法中
end
```

`execute` 会根据 `bin/apisix` 接受到第一个参数来执行预定义的方法。

```lua
local action = {
    help = help,
    version = version,
    init = init,
    init_etcd = etcd.init,
    start = start,
    stop = stop,
    quit = quit,
    restart = restart,
    reload = reload,
    test = test,
}
```

可以在action table 中查看 apisix 支持哪些指令。

在启动流程中，我们重点关注 `start` 方法。

---

### start

`start` 方法核心流程如下：

1. 清理 env

   ```lua
   cleanup(env)
   ```

2. 简单校验
   - 确保 APISIX 不能运行在 /root 文件夹下
   - 确保旧的 APISIX 进程结束运行
3. 处理命令行参数，支持自定义配置文件：

   ```lua
   local parser = argparse()
   parser:option("-c --config", "location of customized config.yaml")
   local args = parser:parse()
   ```

   若提供了 -c 参数，会将自定义配置写入 `profile:customized_yaml_index()`。

4. 初始化环境和 ETCD：

   ```lua
   init(env)
   if env.deployment_role ~= "data_plane" then
       init_etcd(env, args)
   end
   ```

5. 启动 openresty

   ```lua
   util.execute_cmd(env.openresty_args)
   ```

   `util.execute_cmd` 前文已经分析过了，这里就是将 `openresty_args` 作为命令执行。

   最终，APISIX 是通过启动一个 OpenResty 进程来运行的，使用指定的程序目录和 nginx.conf 配置文件。
   
   我们可以通过 `ps -ef` 查看所有运行中的进程
   
   ```bash
   root       25144       1  0 07:17 ?        00:00:00 nginx: master process /usr/bin/openresty -p /workspace -c /workspace/conf/nginx.conf
   nobody     25145   25144  0 07:17 ?        00:00:02 nginx: worker process
   root       25157   25144  0 07:17 ?        00:00:02 nginx: privileged agent process
   ```
   
   可以看到三类nginx相关的进程正在运行，熟悉 nginx 的肯定对这些信息不陌生：
   
   - master 进程：主进程，管控 + 配置管理
   - worker 进程：真正处理流量
   - privileged agent 进程：安全地执行特权操作
   
   可以看到 master 进程输出的信息正是 openresty_args 变量的内容。

---

## 停止服务

分析到这里，停止 APISIX 服务的分析就十分简单了，我们直接找到 `apisix/cli/ops.lua` 文件，里面的 `stop` 方法就是关闭的指令：

```lua
local function stop(env)
    cleanup(env)

    local cmd = env.openresty_args .. [[ -s stop]]
    util.execute_cmd(cmd)
end
```

可以看出只是给 openresty_args 拼接了 `-s stop` 参数，这是 openresty 退出的指令，执行它 openresty 将会有序停止服务。

## 总结

本文从 bin/apisix 脚本入手，逐层分析了 APISIX 的启动逻辑。最终可以看到，APISIX 实际上是通过执行 openresty -p <工作目录> -c <nginx.conf> 来启动一个 OpenResty 进程，再由 master 进程派生出 worker、agent 等子进程，从而承载整个网关服务。

在下一篇文章中，我们会接着探讨：APISIX如何管理自身的配置文件，配置文件是如何被翻译成 OpenResty 可以识别的配置。
