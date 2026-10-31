---
title: 在 Mac 上用 OrbStack 打造零配置的本地 Kubernetes 開發環境
date: 2026-10-31 23:51:15
tags: orbstack kubernetes docker mac mdns
---

> 本文改寫自 Tony.Wu's Blog 的[《在 Mac 上用 OrbStack 打造零配置的本地 Kubernetes 開發環境》](https://www.itnotetk.com/2026/04/10/%E5%9C%A8-mac-%E4%B8%8A%E7%94%A8-orbstack-%E6%89%93%E9%80%A0%E9%9B%B6%E9%85%8D%E7%BD%AE%E7%9A%84%E6%9C%AC%E5%9C%B0-kubernetes-%E9%96%8B%E7%99%BC%E7%92%B0%E5%A2%83/)。

身為一個長期在 Mac 上跑容器和 K8s 的開發者，我一直在尋找更輕量、更順手的工具。之前分享過用 Dnsmasq 處理本地 DNS 解析的做法，雖然堪用，但每次都要手動設定實在有點煩。後來發現了 OrbStack 這個工具，它的 slogan 是「Say goodbye to slow, clunky containers and VMs」，但我最愛的其實是它那個零配置的本地域名解析能力。

今天就來聊聊我實際使用 OrbStack 搭配本地 K8s 環境的心得，包含容器域名、mDNS 原理、內建 HTTPS 支援，以及原生 K8s 支援。

## 容器域名：不用記 Port 的日子

OrbStack 會自動幫每個容器分配一個 `.orb.local` 的域名。舉例來說，如果我在本地起了一個叫 `getting-started` 的容器：

```bash
docker run -d -p 3000:80 --name getting-started docker/getting-started
```

以往我得透過 `localhost:3000` 來存取，但有了 OrbStack，直接在瀏覽器打 `getting-started.orb.local` 就能連上，完全不用指定 Port，非常絲滑。

以下整理容器存取方式的對比：

| 存取方式 | 範例 | 是否需要指定 Port |
| --- | --- | --- |
| 傳統 localhost | `http://localhost:3000` | 是 |
| OrbStack 域名 | `http://getting-started.orb.local` | 否 |
| Docker Compose | `http://service.project.orb.local` | 否 |

## 背後原理：mDNS 組播 DNS

好奇它怎麼做到的？我們可以用幾個指令來驗證。先 ping 看看：

```bash
ping getting-started.orb.local
# PING getting-started.orb.local (192.168.215.3): 56 data bytes
# 64 bytes from 192.168.215.3: icmp_seq=0 ttl=63 time=1.714 ms
```

再用 `docker inspect` 確認容器 IP：

```bash
docker inspect getting-started \
  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# 192.168.215.3
```

完全吻合。重點是 `/etc/hosts` 沒有任何改動，OrbStack 是透過 mDNS（Multicast DNS）來做的。macOS 原生就支援 `.local` TLD 的 mDNS 解析，可以透過 `scutil --dns` 看到相關的 resolver 設定：

```bash
scutil --dns
# resolver #2
#   domain   : local
#   options  : mdns
#   timeout  : 5
```

也能用 `dns-sd` 指令進一步確認：

```bash
dns-sd -G v4v6 getting-started.orb.local
# getting-started.orb.local.  192.168.215.3  300
```

這代表 OrbStack 透過 Apple 的 Bonjour 機制在本機網路上廣播容器的域名與 IP 對應，完全不需要額外的 DNS 伺服器。

## 自訂域名

如果預設的 `container-name.orb.local` 不符合你的需求，OrbStack 也允許自訂域名，透過 Docker Label 注入即可：

```bash
docker run --rm -l dev.orbstack.domains=myapp.local docker/getting-started
```

要注意的是，因為底層是 mDNS，所以自訂域名只能用 `.local` 這個 TLD。

另外，連到 `http://orb.local` 還能看到所有正在運行的容器清單，方便管理。

## 內建 HTTPS 支援

早期要在本地 K8s 跑 HTTPS，得用 mkcert 自簽憑證、手動建立 TLS Secret，再掛到 Ingress 上，步驟相當繁瑣。現在已經不需要這一套了。

OrbStack 在較新版本中已內建 HTTPS 支援並預設啟用，會自動為 `.orb.local` 域名簽發並信任憑證。我們不需要手動建立或信任自簽憑證，容器服務就能直接用 `https://container-name.orb.local` 存取。

想搶先體驗的話，可以到設定中將更新頻道切換為 Canary，然後檢查更新即可。

## OrbStack 原生 K8s 支援

OrbStack 內建了一個輕量的單節點 K8s 叢集，針對本地開發做了最佳化。如果你不需要多節點叢集，其實不用再另外裝 Kind 或 minikube，直接用 OrbStack 內建的就好。

除了域名解析之外，你還可以直接透過 Pod IP 或 Service IP 存取服務，不需要做 `port-forward`，對日常開發測試來說非常方便。

## 傳統做法 vs OrbStack

以下整理傳統 Dnsmasq 方式與 OrbStack 的差異：

| 項目 | 傳統做法（Dnsmasq） | OrbStack |
| --- | --- | --- |
| DNS 設定 | 手動安裝與設定 Dnsmasq | 零配置 |
| 域名格式 | 自定義 | `*.orb.local` 自動分配 |
| TLS 憑證 | 手動建立 + 手動信任 | 內建 HTTPS，自動信任 |
| Port Forwarding | 需要 | 不需要 |
| 適用範圍 | Docker / K8s | Docker / K8s / Linux VM |

## 小結

從 Dnsmasq 到 OrbStack，本地開發環境的體驗有了很大的進步。OrbStack 的零配置域名、mDNS 整合、內建 HTTPS 與原生 K8s 支援，讓我在本地開發時省下不少瑣碎的設定時間。如果你也是在 Mac 上做雲原生開發的人，蠻推薦試試看的。
