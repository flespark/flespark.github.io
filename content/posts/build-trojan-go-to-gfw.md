---
title: "科学上网折腾笔记"
date: 2022-10-27T19:54:03+08:00
draft: false
---

以前基于VMESS的IP直连裸奔配置简单搭建了一个机场，使用的[搬瓦工](https://bandwagonhost.com/)GIA CN2线路，速度和稳定性也都还可以，没有时间捣鼓就这样一直用了一年多。但是几个月前挂机场登陆了一次星际争霸无端被封了半个月，前阵子国庆期间IP又被封了。突然google不能用了还是非常影响工作和学习效率，所以决定花点时间学习网络相关知识，好好升级一下机场配置。技术选型方面，因为IP被封了，也不能等到解封，就只能走Cloudflare的免费CDN了，而且为避免再次被封，选择了伪装性更好的trojan-go协议。相比原先的线路，新的方案流量都要经过CDN代理，原本以为会慢一些，但是实测发现延迟反而有所降低而且跑高速流量更加稳定。trojan-go的流行程度远不如V-project家族产品，网上能找到可靠的文档相对较少，一步步边学习边配置下来也花了不少时间，有必要总结一下整个过程。下次出什么问题时需要重新搭建时也可以作为参考，虽然理论上只有SNI污染导致机场失效换一个域名就能解决的。

### 远端软件配置

远端软件配置的包括购买VPS，域名和对应SSL证书的申请，VPS代理服务以及Cloudflare的配置。VPS就性价比和稳定性而言可以选择搬瓦工，出口带宽和流量基本没有限制，年付240软妹币左右。购买域名选择国外的域名注册商，哪便宜就从那买，[Godaddy](https://godaddy.com/)或者[namecheap](https://www.namecheap.com/)有最便宜的十几块钱第一年。VPS和域名购买网上有许多详细分享就不展开讨论，注册域名完成后将域名解析服务绑定到[Cloudflare](https://www.cloudflare.com/)，这里我使用Godaddy注册的域名。

1. 注册登录Cloudflare后点击右侧"Add a Site"

![202210311056828](/images/202210311056828.png)

2. 输入注册的域名点击"Add site"

![202210311058949](/images/202210311058949.png)

3. 选择最下方的免费方案点击"Continue"

![202210311100259](/images/202210311100259.png)

4. 什么也不用填继续点击"Continue”，提示必须先修改DNS记录，"Confirm"继续

![202210311130236](/images/202210311130236.png)

![202210311131536](/images/202210311131536.png)

5. 显示修改的目标DNS解析服务地址，接下来到Godaddy修改DNS配置

![202210311133520](/images/202210311133520.png)

6. 在Godday域名注册完成之后点击头像->"My Product"，进去点击下方刚刚注册的域名

![202210311112960](/images/202210311112960.png)

![202210311113310](/images/202210311113310.png)

7. 选择"DNS"->"Manage Zones"进入，展开DNS配置

![202210311113007](/images/202210311113007.png)

![202210311116906](/images/202210311116906.png)

8. 往下翻找到找到域名解析服务配置，点击"Change"进去然后选择"Enter my own nameservers"

![202210311119928](/images/202210311119928.png)

![202210311136612](/images/202210311136612.png)

9. 填入Cloudflare提供的DNS解析服务地址，点击"Save"，然后在提示框勾选确认

![202210311149614](/images/202210311149614.png)

![202210311150281](/images/202210311150281.png)

10. 然后右上角popup提示修改成功，DNS解析已转移到Cloudflare管理

![202210311151893](/images/202210311151893.png)

11. 回到原来的Cloadflare界面点击下方的"Done, check nameservers"以继续。域名接入Cloudflare后会要求完成一些安全和流量加速相关的基础配置，这些配置跟机场功能没有什么关系，点击"Get started"后每一步选择默认即可

![202211010910998](/images/202211010910998.png)

![202211010911843](/images/202211010911843.png)

继续Cloudflare配置之前，需要为注册的域名申请一个免费的SSL证书。多数免费的证书申请网站都是通过[Let’s Encrypt](https://letsencrypt.org/)签发的，我这里以[OHTTPS](https://www.ohttps.com/)为例

1. 注册登录OHTTPS之后，在"证书管理"中点击"创建证书"，填入域名下一步

![202210311211923](/images/202210311211923.png)

2. 按要求复制CNMAE记录值，需要添加到Cloudflare的DNS记录中

![202210311212874](/images/202210311212874.png)

3. 回到Cloudflare选择刚配置的域名，在DNS设置中新增CNAME解析记录，"Proxy status"选择"DNS only"，点击"Save"

![202211010917809](/images/202211010917809.png)

4. 返回OHTTPS的验证域名界面，先验证解析记录是否生效然后创建证书（解析记录未生效可以尝试DNS授权模式）

![202211010921151](/images/202211010921151.png)

5. 最后下载生成的3个证书到本地，待会需要上传到VPS

![202211010924472](/images/202211010924472.png)

接着就是在VPS上安装trojan-go的机场服务端软件。机场服务使用新旧哪个Linux发行版基本是没有要求，使用旧的发版本系统对于CPU和内存的占用会相对低一些，但是可能需要手动下载和配置安装包，无法使用BBR加速等新内核才有功能。总的来说对于个人翻墙选新选旧总的来说对于使用体验区别不大，选择口碑好的VPS商家还有氪金能力才更重要。其次VPS的地理位置也比较重要，首先东南亚虽然延迟可选的VPS价格都太高，选择日本浏览Netflix一些资源会受限，美国西海岸城市一般是最好的选择，bwghost的CN2线路价格不高速度还可以，离很多大公司的服务器也比较近。我的VPS的IP被封了，虽然bwghost提供了一个远程登录页面还可以用但是复制粘贴的基本功能也没有，延迟大操作起来很不方便。想到Vultr上使用VPS都是按时计费的，所以我就在Vultr上开了一个临时的VPS作为跳板ssh连接到被封IP的VPS。再次以演示为目的，下面的步骤都是在Vultr部署的Debian 11 VPS上操作的

1.ssh登陆后首先检查发行版默认的防火墙状态，Debain 11使用ufw作为防火墙管理工具，按需打开后面要使用到到的端口

```bash
$ ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)

root@vultr:~# ufw allow 80,443
```

2.从github（或者github镜像站）确认最新trojan-go的release包下载路径，下载解压得到trojan-go可执行文件放到适当路径，安装nginx

```bash
$ wget https://github.com/p4gefau1t/trojan-go/releases/download/v0.10.6/trojan-go-linux-amd64.zip
$ unzip trojan-go-linux-amd64.zip
$ cp trojan-go /usr/bin/

$ apt install nginx -y
```

3.scp上传前面的申请的证书fullchain.cer文件和key文件到VPS，然后添加trojan-go和nginx的配置文件，手动添加trojan-go启动项

```bash

# 添加trojan-go服务端配置文件
$ mkdir -p /etc/trojan-go
$ cat <<-'EOF' >/etc/trojan-go/config.json
> {
  "run_type": "server",
  "log_level": 0,
  "local_addr": "0.0.0.0",
  "local_port": 443,
  "remote_addr": "127.0.0.1",
  "remote_port": 80,
  "password": ["s9cbH8&2m%xN"],
  "ssl": {
    "cert": "/usr/local/etc/ssl_certificates/dot4.xyz.fullchain.cer",
    "key": "/usr/local/etc/ssl_certificates/dot4.xyz.key",
    "fallback_port": 8080,
    "sni": "api.dot4.xyz"
  },
  "websocket": {
    "enabled": true,
    "path": "/passthrough",
    "host": "api.dot4.xyz"
  }
}
EOF

# 添加nginx配置文件
$ cat <<-'EOF' >/etc/nginx/nginx.conf
> events {
    worker_connections 1024;
}

http {
    server {
        listen      80 default_server;
        server_name _;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }

    server {
        listen 8080 http2;
        server_name _;
        return      400;
    }
}
EOF

# 添加trojan-go systemd启动项
$ cat <<'EOF' >/usr/lib/systemd/system/trojan-go.service
> # /usr/lib/systemd/system/trojan-go.service
[Unit]
Description=Trojan-Go - An unidentifiable mechanism that helps you bypass GFW
Documentation=https://p4gefau1t.github.io/trojan-go/
After=network.target nss-lookup.target

[Service]
User=nobody
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ExecStart=/usr/bin/trojan-go -config /etc/trojan-go/config.json
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
EOF

```

上面的trojan-go和nginx均为可以保证良好工作的基础配置，trojan-go配置中一些选项需根据个人情况稍作调整：

- "password"：建立trojna-go连接的自定义密码，配置客户端需要填入相同的密码
- "ssl"->"cert": 上传到VPS的证书文件路径
- "ssl"->"key": 上传到VPS的证书私钥路径
- "ssl"->"fall_back": 遇到非TLS连接时回落端口，由nginx处理返回400即可
- "ssl"->"sni": TLS连接用于检查http请求url是否正确的机制，和域名保持一致
- "websocket"->"enable"：通过CDN代理的trojan-go连接必须打开websocket
- "websocket"->"path"：websocket请求路径，填一个比较复杂的字符串，客户端中需要保持一致
- "websocket"->"host"：使用了CDN，必须填入域名

4.使能和重启服务，确认trojan-go和nginx均正常启动

```bash
$ systemctl restart nginx
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-11-01 05:47:59 UTC; 23min ago
       Docs: man:nginx(8)
   Main PID: 10678 (nginx)
      Tasks: 2 (limit: 2339)
     Memory: 1.6M
        CPU: 6ms
     CGroup: /system.slice/nginx.service
             ├─10678 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─10679 nginx: worker process

Nov 01 05:47:59 vultr systemd[1]: nginx.service: Succeeded.
Nov 01 05:47:59 vultr systemd[1]: Stopped A high performance web server and a reverse proxy server.
Nov 01 05:47:59 vultr systemd[1]: Starting A high performance web server and a reverse proxy server...
Nov 01 05:47:59 vultr systemd[1]: nginx.service: Failed to parse PID from file /run/nginx.pid: Invalid argument
Nov 01 05:47:59 vultr systemd[1]: Started A high performance web server and a reverse proxy server.

$ systemctl enable trojan-go
Created symlink /etc/systemd/system/multi-user.target.wants/trojan-go.service → /lib/systemd/system/trojan-go.service.
$ systemctl start trojan-go
$ systemctl status trojan-go
● trojan-go.service - Trojan-Go - An unidentifiable mechanism that helps you bypass GFW
     Loaded: loaded (/lib/systemd/system/trojan-go.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-11-01 06:10:32 UTC; 8s ago
       Docs: https://p4gefau1t.github.io/trojan-go/
   Main PID: 11065 (trojan-go)
      Tasks: 4 (limit: 2339)
     Memory: 2.7M
        CPU: 6ms
     CGroup: /system.slice/trojan-go.service
             └─11065 /usr/bin/trojan-go -config /etc/trojan-go/config.json

Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/trojan.NewServer:server.go:2>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/mux.NewServer:server.go:92 m>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/simplesocks.NewServer:server>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/websocket.NewServer:server.g>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/trojan.NewServer:server.go:2>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/statistic/memory.NewAuthenticator:m>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/trojan.NewServer:server.go:2>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/mux.NewServer:server.go:92 m>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/simplesocks.NewServer:server>
Nov 01 06:10:32 vultr trojan-go[11065]: [DEBUG] 2022/11/01 06:10:32 github.com/p4gefau1t/trojan-go/tunnel/tls.(*Server).AcceptConn:ser>
```

然后回到Cloudflare添加域名解析

1. 回到Cloudflare网页上，在"SSL/TLS"->"Overview"中选择加密模式，按图中选择"Full(strict)"

   ![202211011734882](/images/202211011734882.png)

2. 接着在DNS设置中新增一条A记录，对应trojan-go中配置的URL路径，保持CDN代理打开状态，点击"Save"

   ![202211011745839](/images/202211011745839.png)

3. 接着尝试在浏览器中输入URL打开，可以看到nginx的默认主页

   ![202211011748000](/images/202211011748000.png)

这样服务端的配置就算完成了，因为每一步都是参照网上容易找到的实现来进行的，其中也有不少可以优化的地方和进阶配置，比如使用cloudflare或者trojan也可以直接申请https证书，VPS内核切换拥塞控制为bbr算法，Cloudflare高级配置，容器部署，这些坑留到后期来填。

### 客户端软件的配置

客户端连接到机场的机制是在本机创建一个HTTP或SOCKS的代理端口，同时更改系统的代理设置，然后各种桌面用户软件在发送网络请求时首先会询问系统是否设置了代理，如果是就将原先的TCP/UDP数据经过HTTP/SOCKS再次包裹发送到代理端口，客户端解出原先的TCP/UDP数据再根据所使用的机场协议再次包裹发往机场（或者CDN等其他代理）。这里就要求客户端能区分请求的网址或IP中，哪些服务器位于中国大陆的网址和IP，哪些是局域网IP，然后将这些地址的请求转回直连。当前主流翻墙软件是配置本地的网址和IP路由规则文件实现的，github存在许多定期更新的仓库路由规则文件的仓库，这里从[v2ray-rules-dat](https://github.com/Loyalsoldier/v2ray-rules-dat)下载的IP路由规则文件geopip.dat和网址规则文件geosite.dat。以下介绍不同系统中断上软件的配置

####	Windows,Linux和MacOS
trojan-go的软件组成非常简单，只有一个二进制文件，根据配置文件既可以工作在服务器模式也可以工作在客户端模式，所以在Windows或者Linux上只要以正确的参数执行tranjon-go文件，并在系统代理中添加代理就能工作，使用桌面客户端[qv2ray](https://github.com/Qv2ray/Qv2ray)（已停止维护）只是节省添加启动脚本的工作。以下是qv2ray的配置步骤

1. 从github下载软件最新的Qv2ray v2.7.0[软件](https://github.com/Qv2ray/Qv2ray/releases/tag/v2.7.0)，只有最后这个版本才支持trojan-go插件。同时下载一个老版本的[v2ray](https://github.com/v2fly/v2ray-core/releases/tag/v4.45.0)（因为qv2ray原本是为v2ray开发的，会首先检查v2ray文件是否存在，最新版的v2ray的接口有变动），trojan-go，Qv2ray的[trojan-go插件](https://github.com/Qv2ray/QvPlugin-Trojan-Go/releases/tag/v3.0.0)，以及路由规则文件

2. 安装好Qv2ray后，选择"Perference"->"Kernel Setting"配置v2ray核心可执行文件完整路径和路由规则文件所在目录

![202211021715380](/images/202211021715380.png)

3. 将下载的trojan-go插件复制到插件目录，不同系统由"Plugins"->"Open Local Plugin Fold"确认，重新打开Qv2ray

![202211021717366](/images/202211021717366.png)

4. 在Qv2ray主界面中点击"Plugins"打开插件管理界面，可以看到已加载的trojan-go插件。在trojan-go插件设置中添加trojan-go可执行文件的完整路径

![202211021723610](/images/202211021723610.png)

5. 剩下的配置就都可以通过如下的json配置文件导入，点击Qv2ray左下角的"Import"，选择"Advanced"->"Open JSON Editor"填入如下内容，点击"OK"导入就完成了一个代理出口配置的创建，注意保持客户端和服务器中相应配置的一致

   ![202211021933411](/images/202211021933411.png)
```json
{
  "outbounds": [
      {
          "_QV2RAY_USE_GLOBAL_FORWARD_PROXY_": false,
          "mux": {
              "concurrency": 1,
              "enabled": null
          },
          "protocol": "trojan-go",
          "sendThrough": "0.0.0.0",
          "settings": {
              "encryption": "",
              "host": "api.dot4.xyz",
              "mux": false,
              "password": "s9cbH8&2m%xN",
              "path": "/passthrough",
              "plugin": "",
              "port": 443,
              "server": "api.dot4.xyz",
              "sni": "api.dot4.xyz",
              "type": 1
          },
          "streamSettings": {
          },
          "tag": "PROXY"
      }
  ]
}

```

6. Qv2ray默认在本地打开`http://127.0.0.1:1089`和`socks5://127.0.0.1:8889`两个代理监听端口，设置代理连接后Linux/MacOS可以通过如下命令测试代理的连通性，返回301表示可以正常访问google了。
   ```bash
   $ curl --proxy socks5://localhost:1089 google.com
   <HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
   <TITLE>301 Moved</TITLE></HEAD><BODY>
   <H1>301 Moved</H1>
   The document has moved
   <A HREF="http://www.google.com/">here</A>.
   </BODY></HTML>
   ```

   Windows则可以利用powershell提供的iwr命令检查（iwr不支持socks5），如下表示正常

   ```powershell
   > Invoke-WebRequest -Proxy http://127.0.0.1:8889 google.com
   
   
   StatusCode        : 200
   StatusDescription : OK
   Content           : <!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="en"><head><meta conten
                       t="Search the world's information, including webpages, images, videos and more. Google has many spe
                       ci...
   RawContent        : HTTP/1.1 200 OK
   ```

   经验证Windows的浏览器此时可以正常通过代理访问Google了，但是Linux上使用Edge没有走代理但是Firefox则是走的代理，可能两个应用使用的网络API不一样。

####	Android

trojan-go是一种最近才流行起来的代理协议，所以相对而言支持的应用比较少，[SagerNet](https://github.com/SagerNet/SagerNet)提供了对trojan-go的完整支持，可以从F-Droid应用商店下载。使用SagerNet手动下载和指定机场协议核心文件和路由规则文件，而会在第一次启动代理时自动下载这些文件并且自动更新，所以只要二维码导入PC客户端已有的代理配置即可建立连接（在Qv2ray中选择一个代理配置点击"Configuration Details"右下侧阴影即可展示分享二维码）。

####	IOS

IOS目前使用较多的代理软件是shadowrocket，支持trojan-go，内置简单的路由规则对于手机应用也完全够用。不过得自行注册一个美区apple ID登录apple store下载，收费2.99刀，在国内电商平台充值3美金的礼品卡即可。websocket的同样支持二维码扫描导入，这里不过多赘叙。

### 软路由透明代理的配置

新版本的鸿蒙系统修改了内核使得所有的代理软件都失效了，我手上有一台华为手机和平板都没法翻墙了，而且Android后台一直挂着翻墙软件耗电增加也比较明显。抱着学习的态度，我入手了一个软路由以实现局域网的全局代理，原本打算使用现成的OpenWRT的固件开代理，但是OpenWRT没有驱动支持我所使用的最新的AX210网卡。所以下面的操作均基于升级了5.19内核的Debian 11，将一个网口作为WAN，3个网口和一个无线AP作为LAN口以实现透明代理。

1. 首先下载trajon-go，geoip.dat和geosite.dat放到合适的目录，添加如下客户端配置文件

   ```bash
   $ echo <<-'EOF' > /etc/trojan-go/config.json
   > {
       "run_type": "client",	//透明代理改为nat
       "local_addr": "127.0.0.1",
       "local_port": 1080,
       "remote_addr": "api.dot4.xyz",
       "remote_port": 443,
       "password": [
           "s9cbH8&2m%xN"
       ],
       "ssl": {
           "sni": "api.dot4.xyz"
       },
       "websocket": {
           "enabled": true,
           "path": "/passthrough",
           "host": "api.dot4.xyz"
       },
       "mux": {
           "enabled": true,
           "concurrency": 8,
           "idle_timeout": 60
       },
       "router": {	//透明代理不支持router需删除
           "enabled": true,
           "bypass": [
               "geoip:cn",
               "geoip:private",
               "geosite:cn",
               "geosite:private"
           ],
           "block": [
               "geosite:category-ads"
           ],
           "proxy": [
               "geosite:geolocation-!cn"
           ],
           "default_policy": "proxy",
           "geoip": "/usr/share/trojan-go/geoip.dat",
           "geosite": "/usr/share/trojan-go/geosite.dat"
       }
   }
   EOF
   ```

   

2. 添加和服务端相同的开机自启配置文件，启动trojan-go服务测试配置的能够正常工作后，修改JSON配置中的"run_type"为"nat"切换为透明代理模模式。重启trojan-go服务报错：`hub.com/p4gefau1t/trojan-go/proxy.(*Option).Handle:option.go:78 router is not allowed in nat mode`，因为trojan使用的tproxy实现的全局代理模式，tproxy无法内部分流。trojan-go也没有内建DNS功能，所以需要额外搭建DNS服务器以实现分流。但是在此之前我们首先要对网口做相关配置，模拟实现一个普通路由器的功能

   ```bash
   $ apt install -y netplan.io
   
   # netplan配置
   # WAN口根据上级网关配置，这里是192.168.1.1/24
   $ cat <<-'EOF' >/etc/netplan/01-ethernet.yaml
   > network:
     version: 2
     renderer: NetworkManager
     ethernets:
       net0:
         dhcp4: yes
         dhcp4-overrides:
           use-dns: no
         addresses: [192.168.1.122/24]
         gateway4: 192.168.1.1
         nameservers:
           search: [lan]
           addresses: [127.0.0.1, 223.5.5.5, 233.6.6.6, 192.168.1.1]
       net1:
         dhcp4: no
         dhcp6: no
       net2:
         dhcp4: no
         dhcp6: no
       net3:
         dhcp4: no
         dhcp6: no
     wifis:
       wifi0:
         dhcp4: no
         dhcp6: no
         access-points:
           "dokodemodoor":
             mode: ap
             password: "123skr~~"
     bridges:
       lan:
         dhcp4: no
         dhcp6: no
         addresses: [10.60.122.254/24]
         interfaces: [net1, net2, net3, wifi0]
   EOF
   
   $ netplan generate	# 生成临时的后端配置
   $ netplan try	# 尝试应用配置
   $ netplan apply	# 应用配置
   $ systemctl enables netplan # 使能服务，下次启动就会自动应用/etc/netplan中的配置
   ```

   上面的配置中主要实现了以下功能：

   - net0~net3对应软路由物理口的[固定网口名称](https://wiki.archlinux.org/title/Network_configuration#Change_interface_name)。其中net0作为WAN口，设置了一个固定IP，同时也接受上级路由dhcp分配的IP

   - net1，net2，net3通过虚拟网桥连接作为LAN口，WIFI网卡配置工作在AP模式，纳入LAN口

   - 设置`dhcp4-overrides`->`use-dns`为no，即不从dhcp获取advertised DNS服务器地址，使软路由的DNS查找顺序为：本机端口，阿里DNS，上级路由器，这样DNS查询首先会由dnsmasq处理。但是非常遗憾的是目前netplan中此项对于NetworkManager作为renderer时不生效，所以只能在NetworkManager配置中指定dnsmas为代理nameserver或者手动修改/etc/resolv.conf并加锁防止NetworkManager更新：

     ```bash
     # 修改NetworkManager配置
     $ sed -i "/\[main\]/a dns=dnsmasq" /etc/NetworkManager/NetworkManager.conf
     
     # 使nameserver 127.0.0.1处于第一的顺序并加锁
     $ chattr +i /etc/resolv.conf
     ```

   这里使用netplan只是完成链路层的部分配置，模拟一个路由器还要添加WAN口和LAN口之间的NAT，以及配置绑定到LAN口的dhcp服务器，在下面的配置中借助iptables和dnsmasq来实现

   

3. DNS分流依靠dnsmasq，dnscrypt-proxy服务，dnsmaq连接到国内的DNS服务器负责正常的网站的DNS查询，dnscrypt-proxy则使用加密协议连接到国外的DNS服务器查询被墙异常网站的IP。所以还是要有一个办法区分国内和国外域名，这里通过ipset保存gfwlist提供的被墙域名目录，然后使用iptables匹配异常IP转发给透明代理端口，国内则直连。首先搭建好DNS服务，顺便使用dnsmasq搭建LAN的dhcpd服务

   ```bash
   # 安装dnsmasq和dnscrypt-proxy，以及dns命令行工具
   $ apt install -y dnsmasq dnscrypt-proxy bind9-dnsutils
   
   # 确认/etc/resolv.conf中本地优先的DNS查找顺序
   $ cat /etc/resolv.conf
   # Generated by NetworkManager
   search lan
   nameserver 127.0.0.1
   nameserver 223.5.5.5
   nameserver 223.6.6.6
   # NOTE: the libc resolver may not support more than 3 nameservers.
   # The nameservers listed below may not be recognized.
   nameserver 192.168.1.1
   
   # 配置dnsmasq
   $ cat <<'EOF' > /etc/dnsmasq.conf
   > no-resolv
   listen-address=::1,127.0.0.1,10.60.122.254	# 对本机和LAN口提供DNS服务
   cache-size=1000
   interface=lan	# 将dhcpd绑定到lan口
   server=233.5.5.5	# 国内使用阿里DNS
   server=233.6.6.6
   expand-hosts
   domain=dokodemodoor.lan
   address=/dokodemodoor.lan/10.60.122.254
   dhcp-range=10.60.122.1,10.60.122.100,255.255.255.0,24h	# dhcpd配置：分配ip范围，租约时长
   EOF
   
   dnsmasq --test # 确认配置文件语法没有问题
   systemcty restart dnsmasq # 重启dnsmasq服务
   
   # dnscrypt-proxy.service使用默认的设置即可
   # 额外配置dnscrypt-proxy.socket，确认配置文件文件中存在listen_address=[]这一行
   $ cat <<'EOF' > /usr/lib/systemd/system/dnscrypt-proxy.socket
   > [Unit]
   Description=dnscrypt-proxy listening socket
   Documentation=https://github.com/DNSCrypt/dnscrypt-proxy/wiki
   Before=nss-lookup.target
   Wants=nss-lookup.target
   Wants=dnscrypt-proxy-resolvconf.service
   
   [Socket]
   ListenStream=127.0.2.1:1053	# 监听端口改为其他未使用的端口，避免与dnsmasq冲突
   ListenDatagram=127.0.2.1:1053
   ListenStream=[::1]:1053
   ListenDatagram=[::1]:1053
   NoDelay=true
   DeferAcceptSec=1
   
   [Install]
   WantedBy=sockets.target
   EOF
   
   systemctl daemon-reload
   systemctl restart dnscrypt-proxy.socket
   
   # 此时执行nslookup查询应该显示走的本地53端口
   $ nslookup baidu.com
   Server:         127.0.0.1
   Address:        127.0.0.1#53
   
   Non-authoritative answer:
   Name:   baidu.com
   Address: 39.156.66.10
   Name:   baidu.com
   Address: 110.242.68.66
   
   # 使用其他设备通过连接LAN口可以自动获取IP，systemctl status dnsmasq也可以看到相关日志
   
   # 然后从github获取gfwlist用于区分被墙网站，这里直接使用可以github可以找到的gfwlist转dnsmasq规则脚本，下载后执行脚本生成dnsmasq可以识别的路由规则。-p 1053表示将规则的网址转发给监听端口1053的dnscrypt-proxy处理，-s gfwlist表示同时加入名为gfwlist的set
   $ git clone https://github.com/cokebar/gfwlist2dnsmasq.git && cd gfwlist2dnsmasq
   $ ./gfwlist2dnsmasq.sh -p 1053 -s gfwlist -o gfwlist.conf
   
   # 大概浏览gfwlist文件确认路由规则的有效性
   # 将生成的conf文件复制到/etc/dnsmasq.d路径下，dnsmasq会自动读取其中的路由规则
   # 然后需要手动创建名为gfwlist的IP set，用于IP分流
   $ ipset create gfwlist iphash
   
   # 此时DNS查询任意被墙的网址，应该返回正确的IP地址
   # dnscrypt-proxy的日志中/var/log/dnscrypt-proxy/query.log可以看到相关查询记录
   $ nslookup facebook.com
   Server:         127.0.0.1
   Address:        127.0.0.1#53
   
   Non-authoritative answer:
   Name:   facebook.com
   Address: 157.240.22.35
   Name:   facebook.com
   Address: 2a03:2880:f131:83:face:b00c:0:25de
   
   # IP set中也可以看到新增的被墙IP，而国内正常访问的IP不会加到set中
   $ ipset -L gfwlist
   Name: gfwlist
   Type: hash:ip
   Revision: 5
   Header: family inet hashsize 1024 maxelem 65536 bucketsize 12 initval 0x9aa08d27
   Size in memory: 240
   References: 0
   Number of entries: 1
   Members:
   157.240.22.35
   ```

   

4. 完成以上配置，WAN和LAN还是2个单端的网段，加下来添加必要的内核路由转发配置。检测所有的LAN流量的目标IP，如果是正常IP则走直连，被墙IP则转发到给本地的透明代理端口

   ```bash
   # 安装相关软件
   $ apt install -y ebtables iptables netfilter-persistent
   
   # 确认开启了系统的IP转发选项
   $ sed -i "s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g" /etc/sysctl.conf
   $ sysctl -p
   
   # 添加iptables配置
   # 所有net0出口流量做SNAT，这样软路由就相当于一个普通的路由器，连接到LAN口的设备也就能通过WAN口上网
   $ iptables -t nat -A POSTROUTING -o net0 -j MASQUERADE
   
   # 新建TROJAN_GO表
   $ iptables -t mangle -N TROJAN_GO
   
   
   # 目标地址匹配gfwlist set的包打上标记，这里的1080对应trojan-go的本地代理端口
   $ iptables -t mangle -A TROJAN_GO -j TPROXY -p tcp -m set --match-set gfwlist dst --on-ip=127.0.0.1 --on-port 1080 --tproxy-mark 0x01/0x01
   $ iptables -t mangle -A TROJAN_GO -j TPROXY -p udp -m set --match-set gfwlist dst --on-ip=127.0.0.1 --on-port 1080 --tproxy-mark 0x01/0x01
   
   # 从LAN口进入的所有TCP/UDP包，跳转TROJAN_GO链
   $ iptables -t mangle -A PREROUTING -p tcp -i lan -j TROJAN_GO
   $ iptables -t mangle -A PREROUTING -p udp -i lan -j TROJAN_GO
   
   # 添加路由，打上标记的包重新进入本地回环
   $ ip route add local default dev lo table 100
   $ ip rule add fwmark 1 lookup 100
   
   # 完成以上配置后，所有LAN口接入的设备就应该可以科学上网了，但是ip,ipset和iptables添加的配置在每次重启后就会丢失，还要保存作为开机设定
   # 借助netfilter-persistent服务自动在关开机时保存和恢复这些配置，只需指定目录添加自定义脚本
   # 针对ipset添加自定义脚本
   $ cat <<'EOF' >/usr/share/netfilter-persistent/plugins.d/10-ipset
   > #!/bin/sh
   
   set -e
   
   RULE_FILE=/etc/iptables/rules.ipset
   
   check_ipset()
   {
       if [ ! -e /sbin/ipset ]; then
           echo "Error: /sbin/ipset not found" >&2
           exit 1
       fi
       if [ ! -x /sbin/ipset ]; then
           echo "Error: /sbin/ipset is not executable file" >&2
           exit 1
       fi
   }
   
   
   load_rules()
   {
       #load ipset rules
       if [ ! -f "${RULE_FILE}" ]; then
           echo "Warning: skipping ipset (no rules to load)"
       else
           /sbin/ipset restore -file "${RULE_FILE}" > /dev/null || return 1
       fi
   }
   
   save_rules()
   {
       touch "${RULE_FILE}"
       chmod 0640 "${RULE_FILE}"
       /sbin/ipset save -file "${RULE_FILE}" > /dev/null || return 1
   }
   
   flush_rules()
   {
       /sbin/ipset destroy
   }
   
   check_ipset
   case "$1" in
   start|restart|reload|force-reload)
       flush_rules
       load_rules
       ;;
   save)
       save_rules
       ;;
   stop)
       # Why? because if stop is used, the firewall gets flushed for a variable
       # amount of time during package upgrades, leaving the machine vulnerable
       # It's also not always desirable to flush during purge
       echo "Automatic flushing disabled, use \"flush\" instead of \"stop\""
       ;;
   flush)
       flush_rules
       ;;
   *)
       echo "Usage: $0 {start|restart|reload|force-reload|save|flush}" >&2
       exit 1
       ;;
   esac
   EOF
   chmod +x /usr/share/netfilter-persistent/plugins.d/10-ipset
   
   # 针对ip route和ip rule添加自定义脚本
   cat <<'EOF' >/usr/share/netfilter-persistent/plugins.d/8-tproxy_iproute
   > #!/bin/sh
   
   set -e
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   
   load_rules()
   {
   	ip route add local default dev lo table 100 || return 1
   	ip rule add fwmark 1 lookup 100
   }
   
   save_rules()
   {
       echo "Warning: skipping ip route (no need to save)"
   }
   
   flush_rules()
   {
       ip route flush table 100 || return 1
       ip rule del fwmark 1
   }
   
   case "$1" in
   start|restart|reload|force-reload)
       load_rules
       ;;
   save)
       save_rules
       ;;
   stop)
       # Why? because if stop is used, the firewall gets flushed for a variable
       # amount of time during package upgrades, leaving the machine vulnerable
       # It's also not always desirable to flush during purge
       echo "Automatic flushing disabled, use \"flush\" instead of \"stop\""
       ;;
   flush)
       flush_rules
       ;;
   *)
       echo "Usage: $0 {start|restart|reload|force-reload|save|flush}" >&2
       exit 1
       ;;
   esac
   EOF
   chmod +x /usr/share/netfilter-persistent/plugins.d/8-tproxy_iproute
   
   # 手动保存确认脚本能正常工作
   $ netfilter-persistent save
   run-parts: executing /usr/share/netfilter-persistent/plugins.d/10-ipset save
   run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables save
   run-parts: executing /usr/share/netfilter-persistent/plugins.d/25-ip6tables save
   run-parts: executing /usr/share/netfilter-persistent/plugins.d/8-tproxy_iproute save
   Warning: skipping ip route (no need to save)
   
   # 重启执行以下命令，前面ip，ipset和iptables设置的所有规则应该可以完全恢复
   $ ipset -L gfwlist
   $ iptables -t mangle -L
   $ iptables -t nat -L
   $ ip route l table 100
   $ ip rule l fwmark 1
   ```

5. 同时也把gfwlist配置的更新添加为定时服务：

   ```sh
   $ cat <<'EOF' >/usr/lib/systemd/system/update-dnsmasq-rule.service
   [Unit]
   Description=Update dnsmasq rule of gfwlist
   Wants=update-dnsmasq-rule.timer
   After=network.target network-online.target systemd-networkd.service NetworkManager.service
   
   [Service]
   Type=oneshot
   ExecStartPre=/bin/rm -f /etc/dnsmasq.d/.gfwlist.conf
   ExecStart=/usr/bin/gfwlist2dnsmasq.sh -p 1053 -s gfwlist -o /etc/dnsmasq.d/.gfwlist.conf
   ExecStartPost=/bin/mv -f /etc/dnsmasq.d/.gfwlist.conf /etc/dnsmasq.d/gfwlist.conf && /usr/bin/systemctl restart dnsmasq
   EOF
   
   $ cat <<'EOF' >/usr/lib/systemd/system/update-dnsmasq-rule.timer
   [Unit]
   Description=Weekly update dnsmasq rule of gfwlist
   Requires=dnsmasq.service
   
   [Timer]
   Unit=update-dnsmasq-rule.service
   OnCalendar=Weekly
   Persistent=true
   
   [Install]
   WantedBy=timer.target
   EOF
   
   $ systemctl enable update-dnsmasq-rule.service update-dnsmasq-rule.timer
   $ systemctl start update-dnsmasq-rule.timer
   ```

   

   完成了以上设置，所有通过LAN口连接到软路由的设备都能自动获取ip并且科学上网。因为AX210的驱动还有点问题，WIFI AP的我没法测试，我最初追求WIFI 6E入手的这张网卡一直没有利用起来。其实如果改为单臂路由的形式话，利用已有的无线路由器，就不需要软路由具有AP的能力，也就可以随便选一个只有一个网口的开发板或者旧电子设备代替软路由，这里再挖一个坑。另外由于trojan-go透明代理实现tproxy的限制，只开一个工作在透明代理模式下的客户端软路自身是无法科学上网的

   **PS：**软路由同时安装docker会导致tproxy代理失效，执行`sysctl -w net.bridge.bridge-nf-call-iptables=0`回退docker对内核参数的修改可回避此问题

   __PPS：__实际2个月体验下虽然非常稳定，但是极少数网站还是会出现打不开或者加载不完整的情况，但是电脑打开qv2ray代理重试，就能够正常加载，因为qv2ray设置的仅对大陆网站直连。本以为是这些网站服务器的IP被墙了，但是关掉qv2ray的代理后，又可以正常加载。具体原因有待进一步分析

###	参考：

1. [Setup Cloudflare CDN protected Trojan-Go with Docker on Ubuntu 20.04 (thematrix.dev)](https://thematrix.dev/setup-cloudflare-cdn-protected-trojan-go-using-docker-on-ubuntu-20-04/)
2. [Network configuration - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/Network_configuration#Change_interface_name)
3. [翻墙 - DNS污染的原理以及应对策略 - DEV Community 👩‍💻👨‍💻](https://dev.to/metalage303/fan-qiang-dnswu-ran-de-yuan-li-yi-ji-ying-dui-ce-lue-3em)
4. [透明代理 | 新 V2Ray 白话文指南 (v2fly.org)](https://guide.v2fly.org/app/transparent_proxy.html#设置步骤)
5. [Netplan](https://netplan.io/reference#dhcp-overrides)
6. [Network bridge - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/network_bridge)
7. [dnsmasq - Debian Wiki](https://wiki.debian.org/dnsmasq)
8. [dnscrypt-proxy - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/Dnscrypt-proxy)
9. [Linux Kernel Documentation / networking / tproxy.txt (mjmwired.net)](https://mjmwired.net/kernel/Documentation/networking/tproxy.txt)
10. [Linux TPROXY - 浮云可记得拜 - 博客园 (cnblogs.com)](https://www.cnblogs.com/gailtang/p/11200388.html)
11. [IPTables: Fun with MARK « bits | andy smith's blog](https://andys.org.uk/bits/2010/01/27/iptables-fun-with-mark/comment-page-1/)
12. [4.8. Routing Tables (linux-ip.net)](http://linux-ip.net/html/routing-tables.html)
13. [树莓派 + V2Ray 配置透明网关 - YFDou](https://www.yfdou.com/archives/raspberrypi-v2ray-tproxy-gateway.html)

