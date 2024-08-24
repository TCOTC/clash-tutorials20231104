---
title: 搭载 sing-boxp 内核配置 DNS 不泄露教程-geodata 方案
description: 此方案适用于 sing-box，搭载 sing-boxp 内核，采用 `geosite` 和 `geoip` 规则搭配 geosite.db 和 geoip.db 路由规则文件
date: 2024-08-22 14:53:20 +0800
categories: [DNS 配置, DNS 防泄漏]
tags: [sing-box, sing-boxp, ShellCrash, geodata, geosite, 进阶, DNS, DNS 泄露]
---

注：
- 1. 此方案彻底防止了 DNS 泄露（针对在规则集内的 `cn` 走国内 DNS 解析；针对在规则集内的 `proxy` 走 `fakeip`；针对不在规则集内的域名由国外 DNS 解析，如果解析结果是国内 IP 则走国内 DNS 解析，否则走 `fakeip`），兼容性高，可放心使用
- 2. 可进入 <https://ipleak.net> 测试 DNS 是否泄露，“DNS Addresses” 栏目下没有中国国旗（因 `ipleak.net` 域名默认走代理），即代表 DNS 没有发生泄露

# 一、 导入路由规则文件
geosite.db 文件须包含 `fakeip-filter`，推荐导入我定制的[路由规则文件](https://github.com/DustinWin/ruleset_geodata?tab=readme-ov-file#%E4%B8%80-geodata-%E8%A7%84%E5%88%99%E9%9B%86%E6%96%87%E4%BB%B6%E8%AF%B4%E6%98%8E)

# 二、 DNS 防泄漏配置（以 ShellCrash 为例）
1. 进入主菜单 -> 2 内核功能设置 -> 2 切换 DNS 运行模式 -> 4 DNS 进阶设置，将“当前基础 DNS”和“PROXY-DNS”都设置为 `null`  
<img src="/assets/img/dns/dns-null.png" alt="ShellCrash 设置" width="60%" />

2. 连接 SSH 后执行 `vi $CRASHDIR/jsons/dns.json`，按一下 Ins 键（Insert 键），修改为如下内容：

```json
{
  "dns": {
    "servers": [
      { "tag": "dns_direct", "address": [ "https://1.12.12.12/dns-query", "https://223.5.5.5/dns-query" ], "detour": "DIRECT" },
      { "tag": "dns_proxy", "address": [ "https://8.8.8.8/dns-query", "https://1.1.1.1/dns-query" ] },
      { "tag": "dns_fakeip", "address": "fakeip" }
    ],
    "rules": [
      { "outbound": "any", "server": "dns_direct" },
      { "clash_mode": "Direct", "query_type": [ "A", "AAAA" ], "server": "dns_direct" },
      { "clash_mode": "Global", "query_type": [ "A", "AAAA" ], "server": "dns_proxy" },
      { "geosite": [ "proxy" ], "query_type": [ "A", "AAAA" ], "server": "dns_fakeip" },
      { "geosite": [ "cn" ], "query_type": [ "A", "AAAA" ], "server": "dns_direct" },
      { "fallback_rules": [ { "geoip": [ "cn" ], "server": "dns_direct" }, { "match_all": true, "server": "dns_fakeip" } ], "server": "dns_proxy" }
    ],
    "final": "dns_direct",
    "strategy": "prefer_ipv4",
    "independent_cache": true,
    "lazy_cache": true,
    "reverse_mapping": true,
    "mapping_override": true,
    "fakeip": {
      "enabled": true,
      "inet4_range": "198.18.0.0/15",
      "inet6_range": "fc00::/18",
      "exclude_rule": {
        "geosite": [ "fakeip-filter" ]
      }
    }
  }
}
```

按一下 Esc 键（退出键），输入英文冒号 `:`，继续输入 `wq` 并回车