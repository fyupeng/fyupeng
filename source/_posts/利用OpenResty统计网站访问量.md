---
title: 利用OpenResty统计网站访问量
top: false
cover: false
toc: true
mathjax: false
date: 2026-03-05 15:34:18
author: fyupeng
img:
coverImg:
password:
summary: 使用OpenResty + ngnx + Lua脚本统计网站访问量
tags:
- MongoDB
- 数据库
categories:
- Java笔记




---

# 系列运维文章目录

> 作者：**fyupeng**
> 技术专栏：☞ [https://github.com/fyupeng](https://github.com/fyupeng)
> 项目地址：☞ [https://github.com/fyupeng/distributed-blog-system-api](https://github.com/fyupeng/distributed-blog-system-api)
> 访问量预览项目效果：☞ [https://www.fyupeng.cn/detail?name=home&articleId=250904CN0FMKT0X4](https://www.fyupeng.cn/detail?name=home&articleId=250904CN0FMKT0X4)


---

效果图：

![效果图](https://i-blog.csdnimg.cn/direct/14fd121a503146f8bb75d45a4aa77ff5.png)



[TOC]



---

# 前言


例如：随着人工智能的不断发展，机器学习这门技术也越来越重要，很多人都开启了学习机器学习，本文就介绍了机器学习的基础内容。

---


# 一、openresty是什么？

OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，集成了大量精良的 Lua 库、第三方模块及常用工具。其核心是通过 LuaJIT 即时编译器扩展 Nginx 的能力，使开发者能够用 Lua 脚本编写高性能的 Web 应用、网关和动态流量处理逻辑。

## 1.核心特点
**高性能：** 基于 Nginx 的事件驱动架构和 LuaJIT 的高效执行，适合高并发场景。
**动态能力：** 通过 Lua 脚本实现动态路由、认证、日志处理等逻辑，无需重启服务。
**丰富生态：** 内置 Lua-resty 系列库（如 HTTP 客户端、Redis/MySQL 驱动），支持自定义模块开发。
**灵活扩展：** 兼容 Nginx 原生配置，可直接复用现有 Nginx 插件。
## 2.典型应用场景
**API 网关：** 实现鉴权、限流、请求转发等逻辑。
**动态 CDN：** 通过 Lua 脚本实时调整缓存策略或内容改写。
**微服务入口：** 聚合后端服务响应，减少客户端请求次数。
**实时监控：** 在流量层嵌入日志采集或性能分析脚本。
## 3.基础示例
以下是一个简单的 Lua 脚本示例，用于在 Nginx 中返回动态内容：


```lua
location /hello {  
    default_type 'text/plain';  
    content_by_lua_block {  
        ngx.say("Hello, OpenResty!")  
    }  
}  
```

## 4.学习资源
官方文档：[https://openresty.org/](https://openresty.org/)
GitHub 仓库：[https://github.com/openresty/openresty](https://github.com/openresty/openresty)
书籍：《OpenResty 最佳实践》

OpenResty 适用于需要兼顾性能与灵活性的中间层开发，尤其适合替代传统编程语言实现的网关或代理服务。

---
# 二、使用步骤
## 1.脚本编写
**ip_counter.lua** 文件内容如下：

```lua
local shell   = require "resty.shell"
local log     = "/usr/local/nginx/logs/access.log"

--[[ 把数字月 → 英文缩写 ]]
local enMonth = {
    "Jan","Feb","Mar","Apr","May","Jun",
    "Jul","Aug","Sep","Oct","Nov","Dec"
}

--[[ 根据传入 date 返回 grep 要用的正则串 及 实际查询范围 ]]
local function build_grep_pat(date)
    -- 1) 完整日期  2025-03-15
    local y,m,d = date:match("^(%d%d%d%d)%-(%d%d)%-(%d%d)$")
    if y then
        return ("%s/%s/%s"):format(d, enMonth[tonumber(m)], y),  "day"
    end
    -- 2) 只有年月  2025-03
    y,m = date:match("^(%d%d%d%d)%-(%d%d)$")
    if y then
        return ("%s/%s"):format(enMonth[tonumber(m)], y), "month"
    end
    -- 3) 只有年  2025
    y = date:match("^(%d%d%d%d)$")
    if y then
        -- 构造  Jan/2025|Feb/2025|...|Dec/2025
        local t = {}
        for _, mon in ipairs(enMonth) do
            t[#t+1] = mon.."/"..y
        end
        return table.concat(t,"|"), "year"
    end
    return nil,"invalid format"
end

--[[ 统计函数 ]]
local function count_ip(date)
    local pat,range = build_grep_pat(date)
    if not pat then return nil,range end

    local cmd
    if range == "year" then
        -- 一整年：egrep  "Jan/2025|Feb/2025|...|Dec/2025"
        cmd = ('egrep "%s" %s | awk "{print \\$1}" | sort -u | wc -l'):format(pat, log)
    else
        -- 一个月或一天
        cmd = ('grep "%s" %s | awk "{print \\$1}" | sort -u | wc -l'):format(pat, log)
    end

    local ok,out,err = shell.run(cmd, nil, 5000)
    if not ok then return nil,err or "shell error" end
    return tonumber(out:match("%d+"))
end

---------------- 入口 ----------------
local date = ngx.var.arg_date
if not date then
    ngx.status=400
    ngx.say('{"msg":"missing ?date=YYYY[-MM][-DD]"}')
    return
end

local n,err = count_ip(date)
if not n then
    ngx.status=500
    ngx.say(('{"msg":"%s"}'):format(err))
    return
end

ngx.header.content_type = "application/json"
ngx.say(('{"date":"%s","ipCount":%d}'):format(date, n))

```

## 2.代理配置

**nginx.conf** 文件内容如下：

```lua
server {
        listen       9999;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        # 示例：使用OpenResty的Lua模块处理请求

        location /hello {
            default_type 'text/plain';
            content_by_lua_block {
                ngx.say("Hello, OpenResty!")
            }
        }


        # 统计 ip 数量
        location /ipCount {
             content_by_lua_file  /usr/local/openresty/nginx/lua/ip_counter.lua;
        }
```

## 3.启动程序

### 3.1启动OpenResty服务

```bash
openresty -p /usr/local/openresty/nginx -c conf/nginx.conf
```

### 3.2验证配置

```bash
openresty -t -p /usr/local/openresty/nginx -c conf/nginx.conf
```

### 3.3热重载配置不中断服务

```bash 
openresty -s reload -p /usr/local/openresty/nginx
```

### 3.4接口调试

浏览器或postman调用接口 [http://localhost:9999/ipCount?date=${YYYY\[-MM\]\[-DD\]}](http://localhost:9999/ipCount)

例如：

date=2025
date=2025-12
date=2025-12-01

---

# 总结

共同学习，与君共勉！