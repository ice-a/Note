```yml
mixed-port: 7890
#---------------------------------------------------#
allow-lan: true
mode: Rule # 路由工作模式：Rule（规则） / Global（全局代理）/ Direct（全局直连）
# 设置日志输出级别 (默认级别：silent，即不输出任何内容，以避免因日志内容过大而导致程序内存溢出）。
# 5 个级别：silent / info / warning / error / debug。级别越高日志输出量越大，越倾向于调试，若需要请自行开启。
log-level: silent
external-controller: '0.0.0.0:9090'

# 节点配置文件统一存放在 ./profiles/proxies/ 目录中

proxy-providers: # 代理集合，代理提供者
# 从链接导入的配置文件（支持多个Clash 订阅链接）
 dogess: # 代理集名称
   type: http # 代理集类型，http订阅链接
   path: ./profiles/proxies/dogess.yaml # 保存路径
   url: https://dogess.work/link/b7zZcRlPfzYxuDKT?clash=1 # clash 订阅链接
   interval: 3600      # 订阅链接自动更新间隔
   # filter: 'Hong Kong' # 删选出含有该关键词的所有节点
   health-check: # latency-test 对代理集包含的结点设置自动延迟测试、测试间隔以及目标网站
     enable: true
     url: http://www.gstatic.com/generate_204
     interval: 300   

# 从链接导入的配置文件（支持多个Clash 订阅链接）
 jkl:
   type: http
   path: ./profiles/proxies/jkl.yaml
   url: https://rss2.jkl-sub.com/link/rOUzj2iYMXySxPd2?clash=1 # clash 订阅链接
   interval: 3600 
   # filter: 'Hong Kong' # 删选出含有该关键词的节点
   health-check:
     enable: true
     url: http://www.gstatic.com/generate_204
     interval: 300

 # 从本地导入的 clash 配置文件
  # bar:
  #   type: file # 代理集类型，file本地配置文件
  #   path: ./profiles/proxies/bar.yaml
  #   health-check:
  #     enable: true
  #     url: http://www.gstatic.com/generate_204
  #     interval: 600

proxy-groups: # 策略组
- name: PROXY # 策略组名称，自己起
  type: url-test # 策略组类型，url-test自动选用延迟最低的节点，规避掉不能正常使用的节点。
  url: http://www.gstatic.com/generate_204 # 节点延迟测试的目标网站
  interval: 3600 # 自动执行“节点延迟测试”的间隔
  use: # 引入proxy-provider 使用 use 关键字
   - dogess
   - jkl
  #  - bar
  proxies:
   - DIRECT

# clash-rules项目提供的规则集
rule-providers: # 规则集合，规则提供者
  reject: # 规则集名称
    type: http # http类型
    behavior: domain # 规则集类型,domain域名
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt" # url地址
    path: ./ruleset/reject.yaml # 保存路径
    interval: 86400 # 自动下载规则集的间隔
  icloud:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/icloud.txt"
    path: ./ruleset/icloud.yaml
    interval: 86400
  apple:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/apple.txt"
    path: ./ruleset/apple.yaml
    interval: 86400
  google:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/google.txt"
    path: ./ruleset/google.yaml
    interval: 86400
  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400
  direct:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400
  private:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/private.txt"
    path: ./ruleset/private.yaml
    interval: 86400
  gfw:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/gfw.txt"
    path: ./ruleset/gfw.yaml
    interval: 86400
  greatfire:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/greatfire.txt"
    path: ./ruleset/greatfire.yaml
    interval: 86400
  tld-not-cn:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/tld-not-cn.txt"
    path: ./ruleset/tld-not-cn.yaml
    interval: 86400
  telegramcidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/telegramcidr.txt"
    path: ./ruleset/telegramcidr.yaml
    interval: 86400
  cncidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    path: ./ruleset/cncidr.yaml
    interval: 86400
  lancidr:
     type: http
     behavior: ipcidr
     url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/lancidr.txt"
     path: ./ruleset/lancidr.yaml
     interval: 86400
# 分流规则
rules:
  - PROCESS-NAME,v2ray,DIRECT
  - PROCESS-NAME,xray,DIRECT
  - PROCESS-NAME,naive,DIRECT
  - PROCESS-NAME,trojan,DIRECT
  - PROCESS-NAME,trojan-go,DIRECT
  - PROCESS-NAME,ss-local,DIRECT
  - PROCESS-NAME,privoxy,DIRECT
  - PROCESS-NAME,leaf,DIRECT
  - PROCESS-NAME,v2ray.exe,DIRECT
  - PROCESS-NAME,xray.exe,DIRECT
  - PROCESS-NAME,naive.exe,DIRECT
  - PROCESS-NAME,trojan.exe,DIRECT
  - PROCESS-NAME,trojan-go.exe,DIRECT
  - PROCESS-NAME,ss-local.exe,DIRECT
  - PROCESS-NAME,privoxy.exe,DIRECT
  - PROCESS-NAME,leaf.exe,DIRECT
  - PROCESS-NAME,Surge,DIRECT
  - PROCESS-NAME,Surge 2,DIRECT
  - PROCESS-NAME,Surge 3,DIRECT
  - PROCESS-NAME,Surge 4,DIRECT
  - PROCESS-NAME,Surge%202,DIRECT
  - PROCESS-NAME,Surge%203,DIRECT
  - PROCESS-NAME,Surge%204,DIRECT
  - PROCESS-NAME,Thunder,DIRECT
  - PROCESS-NAME,DownloadService,DIRECT
  - PROCESS-NAME,qBittorrent,DIRECT
  - PROCESS-NAME,Transmission,DIRECT
  - PROCESS-NAME,fdm,DIRECT
  - PROCESS-NAME,aria2c,DIRECT
  - PROCESS-NAME,Folx,DIRECT
  - PROCESS-NAME,NetTransport,DIRECT
  - PROCESS-NAME,uTorrent,DIRECT
  - PROCESS-NAME,WebTorrent,DIRECT
  - PROCESS-NAME,aria2c.exe,DIRECT
  - PROCESS-NAME,BitComet.exe,DIRECT
  - PROCESS-NAME,fdm.exe,DIRECT
  - PROCESS-NAME,NetTransport.exe,DIRECT
  - PROCESS-NAME,qbittorrent.exe,DIRECT
  - PROCESS-NAME,Thunder.exe,DIRECT
  - PROCESS-NAME,ThunderVIP.exe,DIRECT
  - PROCESS-NAME,transmission-daemon.exe,DIRECT
  - PROCESS-NAME,transmission-qt.exe,DIRECT
  - PROCESS-NAME,uTorrent.exe,DIRECT
  - PROCESS-NAME,WebTorrent.exe,DIRECT
  - DOMAIN,clash.razord.top,DIRECT
  - DOMAIN,yacd.haishan.me,DIRECT
  - RULE-SET,private,DIRECT
  - RULE-SET,reject,REJECT
  - RULE-SET,icloud,DIRECT
  - RULE-SET,apple,DIRECT
  - RULE-SET,google,DIRECT
  - RULE-SET,proxy,PROXY
  - RULE-SET,direct,DIRECT
  - RULE-SET,telegramcidr,PROXY
  - GEOIP,,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
```
