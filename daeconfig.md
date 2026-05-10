#https://github.com/daeuniverse/dae/blob/main/docs/en/configuration/separate-config.md
# Separate Configuration Files

Sometimes you want to break your configuration file into several files. It may be useful in the following cases:

1. You want to switch nodes via modify the config file using tools like `sed`.
2. You copy other's configuration file and you want to overwrite some parts of it.

## Example

Directory Structure:

```sh
# tree /etc/dae
/etc/dae
├── config.d
│  ├── dns.dae
│  ├── node.dae
│  └── route.dae
└── config.dae
```



Config files:

Notes about `include` paths:

- Relative paths (for example `config.d/*.dae`) are resolved relative to the directory of the *entry* config file (the file passed to `dae -c ...`), not relative to the current working directory.
- Absolute paths (for example `/etc/dae/config.d/*.dae`) are used as-is.
- For security reasons, dae only allows included files under the entry config directory.

```jsonc
# config.dae

# load all dae files placed in ./config.d/
include {
    # Relative path example:
    config.d/*.dae

    # Absolute path example:
    /etc/dae/config.d/*.dae
}
global {
    tproxy_port: 12345

    log_level: warn

    tcp_check_url: 'http://cp.cloudflare.com'
    udp_check_dns: 'dns.google:53'
    check_interval: 600s
    check_tolerance: 50ms

    #lan_interface: eth0
    wan_interface: eth0
    allow_insecure: false

    dial_mode: domain
    disable_waiting_network: false
    auto_config_kernel_parameter: true
    sniffing_timeout: 30ms
}
```

```jsonc
# dns.dae
dns {
    upstream {
        # 格式为 scheme://host:port, 支持 tcp/udp/tcp+udp.
        # 如果是host是域名而且有IPv4和IPv6记录, dae 会通过组策略自动选择，例如最小延迟策略
        # 请确保DNS流量通过dae并由dae转发, 这是域路由必须的
        # 如果 dial_mode 是 "ip", 则上游DNS应答不应该被污染, 所以不推荐使用国内公共DNS.
        # alidns: 'udp://223.5.5.5:53'
        # googledns: 'udp://8.8.8.8'

        alidns: 'udp://dns.alidns.com:53'
        googledns: 'tcp+udp://dns.google:53'
    }

    routing {
        # 根据DNS查询的请求，决定使用哪个DNS服务器
        # 规则从上到下匹配
        request {
            # 内置出站:asis,reject
            # 可用的方法qname, qtype
            # 广告拒绝
            qname(geosite:category-ads-all) -> reject
            # 这里的意思是google中是cn的域名使用alidns
            # qname(geosite:google@cn) -> mosdns
            # qname(geosite:cn) -> mosdns
            # 匹配后缀，匹配关键字
            # qname(suffix: abc.com, keyword: google) -> googledns
            # 全匹配和正则匹配
            # qname(full: ok.com, regex: '^yes') -> googledns
            # DNS 请求类型
            # ipv4和ipv6请求使用alidns
            # qtype(a, aaaa) -> googledns
            # cname请求googledns
            # qtype(cname) -> googledns
            # 默认DNS服务器
            qname(geosite:category-ads) -> reject
            qname(geosite:category-ads-all) -> reject
            fallback: alidns
        }
        response {
            upstream(googledns) -> accept
            !qname(geosite:cn) && ip(geoip:private) -> googledns
            fallback: accept
        }

        # 根据DNS查询的响应，决定接受或者使用另外一个DNS服务器重新查询记录
        # 规则从上到下匹配
        # response {
            # 内置出站：accept,reject
            # 可用的方法：qname, qtype, upstream, ip.
            # 如果是发送到googledns的请求响应，则接受，可用于避免循环
            # upstream(googledns) -> accept
            # 意思是如果请求的域名不是国内网站，但是返回了一个私有的IP，那就是被污染了。重新通过googledns请求
            # ip(geoip:private) && !qname(geosite:cn) -> googledns
            # 以上不匹配，默认
            # 反诈劫持重新使用googledns匹配
            # ip(61.160.148.90) -> googledns
            # fallback: accept
        # }
    }
}
```

```jsonc
# node.dae
node {
    node1: 'xxx'
    node2: 'xxx'
}

subscription {
    my_sub: 'https://www.example.com/subscription/link'
}

group {
    my_group {
        filter: subtag(my_sub) && !name(keyword: 'ExpireAt:')
        policy: min_moving_avg
    }

    local_group {
        filter: name(node1, node2)
        policy: fixed(0)
    }
}
```

```jsonc
# route.dae
routing {
    pname(NetworkManager) -> direct
    # Network managers in localhost should be direct to avoid false negative network connectivity check when binding to
    # WAN.
    pname(NetworkManager, systemd-resolved, dnsmasq) -> must_direct
    pname(tailscaled) -> direct
    pname(mosdns) -> must_direct

    # clash 代理客户端直连，防止网络回环
    pname(clash) -> must_direct
    pname(clash-meta) -> must_direct
    pname(mihomo) -> must_direct
    pname(qemu-system-x86) -> must_direct

    # Put it in the front to prevent broadcast, multicast and other packets that should be sent to the LAN from being
    # forwarded by the proxy.
    # "dip" means destination IP.
    dip(224.0.0.0/3, 'ff00::/8') -> direct
    dip(10.0.0.0/24) -> direct

    # 对使用GoogleDNS的解析请求发往远端进行解析
    dip(8.8.8.8) -> proxy
    dip(8.8.4.4) -> proxy
    domain(full: dns.google) -> proxy

    # This line allows you to access private addresses directly instead of via your proxy. If you really want to access
    # private addresses in your proxy host network, modify the below line.
    dip(geoip:private) -> direct
    domain(geosite:gfw) -> proxy
    domain(geosite:category-ads) -> block

    ### qBitorrent Driect
    dscp(4) -> direct

    ### telegram
    dip(geoip:telegram) -> proxy

    ### WX DOMAIN
    domain(suffix: apd-pcdnwxlogin.teg.tencent-cloud.net) -> direct
    domain(suffix: btrace.qq.com) -> direct
    domain(suffix: dldir1.qq.com) -> direct
    domain(suffix: slife.xy-asia.com) -> direct
    domain(suffix: soup.v.qq.com) -> direct
    domain(suffix: vweixinf.tc.qq.com) -> direct
    domain(suffix: weixin110.qq.com) -> direct
    domain(suffix: wup.imtt.qq.com) -> direct
    domain(suffix: iot-tencent.com) -> direct
    domain(suffix: lbs.gtimg.com) -> direct
    domain(suffix: map.qq.com) -> direct
    domain(suffix: qlogo.cn) -> direct
    domain(suffix: qpic.cn) -> direct
    domain(suffix: servicewechat.com) -> direct
    domain(suffix: tenpay.com) -> direct
    domain(suffix: up-hl.3g.qq.com) -> direct
    domain(suffix: vweixinthumb.tc.qq.com) -> direct
    domain(suffix: wechat.com) -> direct
    domain(suffix: wechatlegal.net) -> direct
    domain(suffix: wechatos.net) -> direct
    domain(suffix: wechatpay.com) -> direct
    domain(suffix: weixin.com) -> direct
    domain(suffix: weixin.qq.com) -> direct
    domain(suffix: weixinbridge.com) -> direct
    domain(suffix: weixinsxy.com) -> direct
    domain(suffix: wx.gtimg.com) -> direct
    domain(suffix: wx.qq.com) -> direct
    domain(suffix: wxapp.tc.qq.com) -> direct
    domain(suffix: wxs.qq.com) -> direct
    domain(suffix: yun-hl.3g.qq.com) -> direct

    ### Github
    domain(suffix: github.com) -> proxy
    domain(suffix: githubusercontent.com) -> proxy
    domain(suffix: github.io) -> proxy

    ### Docker
    domain(suffix: docker.io) -> proxy
    domain(suffix: docker.com) -> proxy

    ### DIRECT MACHINE
    
    ### geosite ip cn
    domain(geosite:cn) -> direct
    domain(geosite:china-list) -> direct
    dip(geoip:cn) -> direct
    fallback: proxy

}
```

Then run `dae` via:

```sh
dae run -c /etc/dae/config.dae
```
