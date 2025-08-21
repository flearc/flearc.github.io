---
title: Apache APISIX：配置管理
date: 2025-08-18 17:26:00 +0800
categories: [TEAHICAL, GATEWAY]
tags: [apisix]
---

## 前言

在[上一篇文章](/posts/APISIX_START_AND_STOP)中，我们分析了 APISIX 的启动流程，了解到它本质上是通过启动一个 openresty 进程，并由 nginx worker 来处理请求。

如果熟悉 nginx/openresty，应该知道 nginx 的配置文件是遵循特定语法的静态文本；而 APISIX 则提供了更加友好的 YAML 配置方式，这两种配置风格如何衔接，正是本文要探讨的核心。

接下来，我们将聚焦于 APISIX 的配置管理机制，分析它如何读取 YAML 配置文件，并将其转换为 nginx 可识别的 nginx.conf 配置文件，最终驱动网关正常运行。

## 配置文件

在启动流程中，我们知道 APISIX 实际是通过以下类似命令运行的：

```bash
/usr/bin/openresty -p /workspace -c /workspace/conf/nginx.conf
```

其中 `-c` 参数指定了 nginx 的配置文件。既然如此，我们不妨回到 devcontainer 开发环境，查看一下 conf 目录下的内容：

```bash
conf
|-- apisix.uid          # 运行时生成
|-- apisix.yaml
|-- cert
|   |-- ssl_PLACE_HOLDER.crt
|   `-- ssl_PLACE_HOLDER.key
|-- config.yaml
|-- config.yaml.example
|-- debug.yaml
|-- mime.types
`-- nginx.conf          # 运行时生成
```

值得注意的是，源码仓库中的 conf/ 目录本身只包含若干模板文件，例如 config.yaml、apisix.yaml、debug.yaml 等。

而在 APISIX 启动之后，这个目录会**额外生成**两个文件：

- nginx.conf：运行时生成的 nginx 配置文件
- apisix.uid：记录运行时的信息（与本文无关）

那么问题来了：**nginx.conf 是在什么时候、由谁生成的？**

### nginx.conf

结合上一篇文章的启动流程，可以推断 nginx.conf 一定是在 start 命令执行完成之前生成的。

事实上，答案非常直观：在 apisix/cli/ops.lua 文件中（或者输入命令 ./bin/apisix help）有一个帮助信息，我们可以看到：

```lua
local function help()
    print([[
Usage: apisix [action] <argument>

help:       print the apisix cli help message
init:       initialize the local nginx.conf
init_etcd:  initialize the data of etcd
start:      start the apisix server
stop:       stop the apisix server
quit:       stop the apisix server gracefully
restart:    restart the apisix server
reload:     reload the apisix server
test:       test the generated nginx.conf
version:    print the version of apisix
]])
end
```

这里明确写着：init 命令负责初始化本地的 nginx.conf。

而在 start 命令里，在执行 openresty_args 之前，也会调用一次 init 方法：

```lua
local function start(env, ...)
    -- ... 省略部分代码

    init(env)
 
    if env.deployment_role ~= "data_plane" then
        init_etcd(env, args)
    end

    util.execute_cmd(env.openresty_args)
end
```

这样一来，就进一步验证了我们的推断：nginx.conf 的生成发生在 start 执行完成之前，由 init 负责完成。

因此，要弄清楚 nginx.conf 是如何生成的，我们只需要深入分析 init 方法的逻辑即可。

### init

`init` 方法的逻辑比较长，我们只聚焦关键部分：

```lua
local ngx_tpl = require("apisix.cli.ngx_tpl")
local template = require("resty.template")

local function init(env)
    -- ... 省略部分代码

    -- 读取 YAML 配置
    local yaml_conf, err = file.read_yaml_conf(env.apisix_home)
    if not yaml_conf then
        util.die("failed to read local yaml config of apisix: ", err, "\n")
    end

    -- 校验配置合法性
    local ok, err = schema.validate(yaml_conf)
    if not ok then
        util.die(err, "\n")
    end

    -- 构建 sys_conf，
    --（这里只展示 apisix 和 nginx_config 字段的合并逻辑，其余省略）
    local sys_conf = { -- ... 省略部分代码 }
 
  	for k,v in pairs(yaml_conf.apisix) do
        sys_conf[k] = v
    end
    for k,v in pairs(yaml_conf.nginx_config) do
        sys_conf[k] = v
    end
  
    --- 渲染并写入 nginx.conf
    local conf_render = template.compile(ngx_tpl)
    local ngxconf = conf_render(sys_conf)

    local ok, err = util.write_file(env.apisix_home .. "/conf/nginx.conf",
                                    ngxconf)
    if not ok then
        util.die("failed to update nginx.conf: ", err, "\n")
    end
end
```

从中可以看到，init 方法除了必要的校验外，主要做了以下几步：

1. **读取配置**：从 apisix_home 下加载 YAML 文件。
2. **校验配置**：通过 schema.validate 保证配置合法性。
3. **构建配置表**：将 YAML 里的配置字段合并到 sys_conf，这是 init 中的主要逻辑。
4. **模板渲染**：使用 resty.template 将 sys_conf 渲染进 ngx_tpl 模板。
5. **生成文件**：将渲染结果写入 conf/nginx.conf。

可以看出最关键的就是模版渲染这一步，APISIX 使用了  resty.template，这是来自 [lua-resty-template](https://github.com/bungle/lua-resty-template) 的方法，由它来完成生成 nginx.conf 的工作。

> APISIX 的所有第三方依赖库，可以在源码根目录下的 apisix-master-0.rockspec 文件中找到。
{: .prompt-tip } 


### 模版引擎：lua-resty-template

在看 ngx_tpl 模板之前，我们先理解 template.compile 的作用。它会生成一个 **渲染函数**，调用时传入一个 table 即可完成模板替换。

我们可以在 devcontainer 中新增一个示例文件 template.lua：

```lua
package.path = "/workspace/deps/share/lua/5.1/?.lua;" .. package.path

local template = require("resty.template")

{% raw %}
local func = template.compile([[here is template render message: {{message}}]])
{% endraw %}

print(func{ message = "Hello"})
print(func{ message = "Template"})
```
{: file='template.lua'}

运行命令：

```bash
/usr/local/openresty/luajit/bin/luajit template.lua
```

输出结果为：

```bash
here is template render message: Hello
here is template render message: Template
```

可见 template.compile 会把字符串模板转换为一个可调用的函数，模板中的 
\{\{ ... \}\}
表示待替换的变量，调用时只需传入一个包含对应变量的 table 即可完成渲染。

理解了这一点后，再回到 init 方法就很容易看出：

- ngx_tpl 就是模板；
- sys_conf 就是传入的 table，其中包含所有需要替换的变量及值。

> Lua-resty-template 常见的渲染语法有
>
> - \{\{ var \}\}：输出变量，自动 html 转义
> - { * var * }：输出变量
> - { % lua code % }：插入 Lua 逻辑 （if/for 等）
>
> APISIX 实际上在模版中使用后两种，更多内容可以看 [template syntax](https://github.com/bungle/lua-resty-template?tab=readme-ov-file#template-syntax)。
{: .prompt-tip } 

### ngx_tpl

在 APISIX 的 apisix/cli/ngx_tpl.lua 文件中，我们能看到用于生成 nginx.conf 的模板内容。它几乎就是一份 nginx 配置文件，只是里面穿插了模板语法：

```lua
# Configuration File - Nginx Server Configs
# This is a read-only file, do not try to modify it.
{% raw %}
{% if user and user ~= '' then %}
user {* user *};
{% end %}
{% endraw %}
master_process on;
  
worker_processes {* worker_processes *};
-- 篇幅有限，仅展示部分内容
```

这种设计使得 APISIX 能用 YAML 来描述配置，再通过 ngx_tpl 动态生成标准的 nginx 配置，抹平了不同配置格式的差异，从而实现灵活的配置管理。

## 读取配置

知道了 APISIX 是如何生成 nginx.conf 的，我们再回头来看看 APISIX 是如何读取配置的。下面是 init 中读取 YAML 的代码

```lua
local yaml_conf, err = file.read_yaml_conf(env.apisix_home)
```

read_yaml_conf 核心实现在 apisix/cli.file.lua 文件下。它主要做了以下三件事情：

1. 定位配置文件位置
2. 支持环境变量替换配置
3. 解析并合并默认配置

### 定位配置文件位置

```lua
local local_conf_path = profile:customized_yaml_path()
if not local_conf_path then
    local_conf_path = profile:yaml_path("config")
end
```

1. 其中 customized_yaml_path 是 ./bin/apisix start 命令传入的 -c/--config 参数，如果没有提供自定义路径，默认读取的是 profile:yaml_path("config") 指向的文件

2. 而 yaml_path 还会读取环境变量 APISIX_PROFILE 的值：

```lua
local _M = {
      version = 0.1,
      profile = os.getenv("APISIX_PROFILE") or "",
      apisix_home = (ngx and ngx.config.prefix()) or ""
  }
  
  --
  --  Get yaml file path by filename under the `conf/`.
  --
  -- @function core.profile.yaml_path
  -- @tparam self self The profile module itself.
  -- @tparam string file_name Name of the yaml file to search.
  -- @treturn string The path of yaml file searched.
  -- @usage
  -- local profile = require("apisix.core.profile")
  -- ......
  -- -- set the working directory of APISIX
  -- profile.apisix_home = env.apisix_home .. "/"
  -- local local_conf_path = profile:yaml_path("config")
  function _M.yaml_path(self, file_name)
      local file_path = self.apisix_home  .. "conf/" .. file_name
      if self.profile ~= "" and file_name ~= "config-default" then
          file_path = file_path .. "-" .. self.profile
      end
  
      return file_path .. ".yaml"
  end
```

{: file='apisix/core/profile.lua'}

**实际效果**：
- 如果 `APISIX_PROFILE=dev`，读取 `config-dev.yaml`
- 如果 `APISIX_PROFILE=prod`，读取 `config-prod.yaml`  
- 如果未设置，读取 `config.yaml`


### 支持环境变量替换配置

YAML 配置文件支持使用环境变量，语法为：

{% raw %}
```yaml
key_name: ${{ENVIRONMENT_VARIABLE_NAME:=VALUE}}
```
{% endraw %}

其中 ENVIRONMENT_VARIABLE_NAME 为环境变量名称，VALUE为默认值

这是由 resolve_conf_var 方法实现的，具体实现不是本文的重点，读者可以自行查看。

### 合并用户配置与默认配置

APISIX 本身基于自己的最佳实践，提供了默认配置，我们无需填写每一行配置，只需根据自身需要修改某些配置即可。

默认配置在 apisix/cli/config.lua 文件中，也可以查看 conf/config.yaml.example。

```lua
local default_conf = require("apisix.cli.config")
```

第一步读取的配置文件配置项会覆盖默认配置中的配置项，如果未修改则保持默认配置。

## 总结

本文我们梳理了 APISIX 的配置管理机制，核心要点如下：

- **配置入口是 YAML 文件**
- **运行时 Nginx 配置并非静态文件**，而是通过 init 方法调用 lua-resty-template 渲染 ngx_tpl 生成。
- **读取配置时**，APISIX 会支持环境变量替换，并与默认配置合并，保证缺省项有合理值。





