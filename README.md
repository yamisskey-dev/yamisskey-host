# yamihost

## Service Architecture & Deployment Overview

```mermaid
graph TB
    %% Style definitions
    classDef homeServer fill:#e2e8f0,stroke:#334155,stroke-width:2px
    classDef service fill:#f8fafc,stroke:#64748b,stroke-width:1px
    classDef rpi fill:#fde68a,stroke:#d97706,stroke-width:2px
    classDef proxy fill:#e0e7ff,stroke:#3730a3,stroke-width:2px
    classDef cloudflare fill:#f0fdfa,stroke:#0f766e,stroke-width:1.5px
    classDef security fill:#fee2e2,stroke:#991b1b,stroke-width:1px
    classDef storage fill:#dcfce7,stroke:#16a34a,stroke-width:2px
    classDef local fill:#86efac,stroke:#16a34a,stroke-width:3px
    classDef activitypubStyle fill:#f3e8ff,stroke:#7c3aed,stroke-width:2px
    classDef nostrStyle fill:#fed7aa,stroke:#ea580c,stroke-width:2px
    classDef matrixStyle fill:#dbeafe,stroke:#1d4ed8,stroke-width:2px
    classDef appsStyle fill:#fef3c7,stroke:#ca8a04,stroke-width:2px
    classDef monitoring fill:#d1fae5,stroke:#047857,stroke-width:2px
    classDef iac fill:#f0f9ff,stroke:#0369a1,stroke-width:2px
    
    %% Main Infrastructure
    subgraph main_servers[Main Servers]
        direction LR
        
        subgraph balthasar[balthasar - 本番環境<br/>ユーザー向けサービス専用]
            direction TB
            cloudflared_b[Cloudflared]:::cloudflare
            nginx_b[Nginx + ModSecurity<br/>Reverse Proxy]:::proxy
            
            subgraph activitypub[ActivityPub]
                yui[Misskey Bot]:::service
                yamisskey[Misskey]:::service
                neoquesdon[Neo-Quesdon]:::service
            end
            
            subgraph matrix[Matrix]
                synapse[Synapse]:::service
                element[Element]:::service
            end
            
            subgraph apps[Apps]
                outline[Outline]:::service
                cryptpad[CryptPad]:::service
                minio[MinIO]:::storage
            end
        end
        
        subgraph caspar[caspar - インフラ・監視・セキュリティ基盤]
            direction TB
            cloudflared_c[Cloudflared]:::cloudflare
            nginx_c[Nginx + ModSecurity<br/>Reverse Proxy]:::proxy
            
            subgraph monitoring_services[監視・IaC]
                prometheus[Prometheus]:::monitoring
                grafana[Grafana]:::monitoring
                uptime[Uptime Kuma]:::monitoring
                alertmgr[AlertManager]:::monitoring
                terraform[Terraform]:::iac
                ansible[Ansible]:::iac
            end
            
            subgraph auth_services[認証・セキュリティ]
                authentik[Authentik]:::security
                mcaptcha[mCaptcha]:::security
            end
            
            subgraph nostr[Nostr - 実験系]
                nostream[Nostream]:::service
                rabbit[Rabbit]:::service
            end
        end
        
        subgraph raspberrypi[raspberrypi - Minecraft専用<br/>NVMe SSD 2TB, 8GB RAM]
            direction TB
            playig[playit.gg]:::service
            
            subgraph games[Games]
                minecraft[Minecraft Java<br/>6GB RAM]:::service
            end
        end
        
        internet((Internet)):::cloudflare
    end
    
    %% Cloudflared to MinIO connections
    nginx_b --> minio

    %% Cross-server authentication connections (via Tailscale)
    outline -.->|Tailscale| authentik
    yamisskey -.->|Tailscale| mcaptcha
    
    %% Matrix internal connections
    element --> synapse
    
    %% Minecraft connections
    minecraft --> playig
    
    %% Monitoring connections
    prometheus --> grafana
    
    %% Cloudflared to Nginx connections
    cloudflared_b --> nginx_b
    cloudflared_c --> nginx_c
    
    %% Nginx to services - balthasar
    nginx_b --> yamisskey
    nginx_b --> neoquesdon
    nginx_b --> element
    nginx_b --> synapse
    nginx_b --> outline
    nginx_b --> cryptpad
    
    %% Nginx to services - caspar
    nginx_c --> prometheus
    nginx_c --> grafana
    nginx_c --> uptime
    nginx_c --> authentik
    nginx_c --> mcaptcha
    nginx_c --> nostream
    nginx_c --> rabbit
    
    %% External connections
    playig --> internet
    cloudflared_b --> internet
    cloudflared_c --> internet
    
    %% Apply styles to subgraphs
    class balthasar,caspar homeServer
    class raspberrypi rpi
    class activitypub activitypubStyle
    class nostr nostrStyle
    class matrix matrixStyle
    class apps appsStyle
    class games service
    class auth_services security
    class monitoring_services monitoring
    class cloudflared_b,cloudflared_c cloudflare
    class nginx_b,nginx_c proxy
```

## Proxmox Virtualization Platform & Security Environment

```mermaid
graph TB
    %% Style definitions
    classDef homeServer fill:#e2e8f0,stroke:#334155,stroke-width:2px
    classDef service fill:#f8fafc,stroke:#64748b,stroke-width:1px
    classDef monitoring fill:#d1fae5,stroke:#047857,stroke-width:1px
    classDef security fill:#fee2e2,stroke:#991b1b,stroke-width:1px
    classDef storage fill:#f3e8ff,stroke:#7e22ce,stroke-width:1.5px
    classDef network fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef ctf fill:#fef3c7,stroke:#d97706,stroke-width:2px
    
    %% Proxmox Host
    subgraph proxmox["GMKtec NucBox K10 - Proxmox VE<br/>Core i9-13900HK, 64GB DDR5, 1TB NVMe"]
       %% Storage Configuration
       subgraph storage["Storage Pools"]
           local["local<br/>ISO, Templates, Backups"]:::storage
           local_lvm["local-lvm<br/>VM Disks"]:::storage
       end
       
       %% Network Bridges
       subgraph networks["Virtual Networks"]
           vmbr0["vmbr0 - External<br/>WAN Interface"]:::network
           vmbr1["vmbr1 - Internal LAN<br/>10.0.0.0/24"]:::network
           vmbr2["vmbr2 - DMZ<br/>192.168.100.0/24"]:::network
           vmbr3["vmbr3 - Management<br/>172.16.0.0/24"]:::network
       end
       
       %% Virtual Machines
       subgraph vms["Virtual Machines"]
           subgraph pfsense_vm["pfSense VM - 4c/8GB/32GB"]
               pfsense["pfSense 2.7+"]:::security
               haproxy["HAProxy"]:::service
               openvpn["OpenVPN"]:::security
           end
           
           subgraph tpot_vm["T-Pot VM - 8c/24GB/200GB"]
               tpot["T-Pot 24.04+"]:::security
               cowrie["Cowrie SSH Honeypot"]:::security
               dionaea["Dionaea Multi-protocol"]:::security
               elasticpot["ElasticPot"]:::security
               kibana_tpot["Kibana Dashboard"]:::monitoring
           end
           
           subgraph malcolm_vm["Malcolm VM - 12c/32GB/500GB"]
               malcolm["Malcolm"]:::monitoring
               elasticsearch["Elasticsearch"]:::monitoring
               logstash["Logstash"]:::monitoring
               zeek["Zeek Network Analysis"]:::monitoring
               suricata_malcolm["Suricata IDS"]:::security
               kibana_malcolm["Kibana Analytics"]:::monitoring
           end
           
           subgraph ctf_vm["CTF VM - 4c/8GB/100GB"]
               ctfd["CTFd<br/>競技プラットフォーム"]:::ctf
               challenge_containers["Challenge Containers<br/>Docker隔離環境"]:::ctf
               ctf_web["CTF Web Challenges"]:::ctf
               ctf_pwn["Pwn/Reversing Challenges"]:::ctf
           end
       end
    end
    
    %% Network connections
    vmbr0 --> pfsense_vm
    vmbr2 --> tpot_vm
    vmbr2 --> malcolm_vm
    vmbr2 --> ctf_vm
    vmbr3 --> pfsense_vm
    
    %% Internal VM connections
    pfsense --> vmbr1
    pfsense --> vmbr2
    pfsense --> vmbr3
    
    %% Storage connections
    local --> vms
    local_lvm --> vms
    
    %% Apply styles
    class proxmox homeServer
    class pfsense_vm,tpot_vm,malcolm_vm security
    class ctf_vm ctf
```

## Infrastructure as Code & Automation Systems

```mermaid
graph TB
    %% Style definitions
    classDef iac fill:#f0f9ff,stroke:#0369a1,stroke-width:2px
    classDef ansible fill:#ee0000,stroke:#cc0000,stroke-width:2px,color:#ffffff
    classDef terraform fill:#7b42bc,stroke:#5c32a8,stroke-width:2px,color:#ffffff
    classDef target fill:#e2e8f0,stroke:#334155,stroke-width:1.5px
    classDef proxmox fill:#e86c00,stroke:#cc5500,stroke-width:2px,color:#ffffff
    classDef git fill:#24292e,stroke:#1b1f23,stroke-width:2px,color:#ffffff
    classDef state fill:#fef3c7,stroke:#d97706,stroke-width:1.5px
    
    subgraph iac_hub["caspar - IaC Hub"]
        direction TB
        
        subgraph ansible_system["Ansible Automation"]
            ansible_core["Ansible Core"]:::ansible
            inventory["Inventory<br/>ホスト定義"]:::iac
            playbooks["Playbooks<br/>構成定義"]:::iac
            roles["Roles<br/>再利用可能モジュール"]:::iac
            vault["Ansible Vault<br/>機密情報暗号化"]:::iac
        end
        
        subgraph terraform_system["Terraform IaC"]
            terraform_core["Terraform"]:::terraform
            tf_state["State File<br/>R2 Backend"]:::state
            tf_providers["Providers<br/>Proxmox/Cloudflare"]:::iac
            tf_modules["Modules<br/>VM/Network定義"]:::iac
        end
        
        git_repo["Git Repository<br/>バージョン管理"]:::git
    end
    
    subgraph managed_infra["Managed Infrastructure"]
        direction TB
        
        subgraph physical["物理サーバー (Ansible管理)"]
            balthasar_target["balthasar<br/>本番サービス"]:::target
            caspar_target["caspar<br/>監視・IaC"]:::target
            truenas_target["TrueNAS<br/>ストレージ"]:::target
            rpi_target["Raspberry Pi<br/>Minecraft"]:::target
        end
        
        subgraph virtual["仮想環境 (Terraform管理)"]
            proxmox_api["Proxmox API"]:::proxmox
            proxmox_vms["VMs<br/>pfSense/T-Pot/Malcolm/CTF"]:::target
        end
        
        subgraph cloud["クラウド (Terraform管理)"]
            cf_workers["Cloudflare Workers"]:::target
            cf_pages["Cloudflare Pages"]:::target
            cf_dns["Cloudflare DNS"]:::target
            linode_vps["Linode VPS"]:::target
        end
    end
    
    %% Ansible connections
    ansible_core --> inventory
    ansible_core --> playbooks
    ansible_core --> roles
    playbooks --> vault
    
    ansible_core -->|"SSH"| balthasar_target
    ansible_core -->|"SSH"| caspar_target
    ansible_core -->|"SSH"| truenas_target
    ansible_core -->|"SSH"| rpi_target
    
    %% Terraform connections
    terraform_core --> tf_providers
    terraform_core --> tf_modules
    terraform_core --> tf_state
    
    terraform_core -->|"API"| proxmox_api
    proxmox_api --> proxmox_vms
    terraform_core -->|"API"| cf_workers
    terraform_core -->|"API"| cf_pages
    terraform_core -->|"API"| cf_dns
    terraform_core -->|"API"| linode_vps
    
    %% Git connections
    git_repo --> ansible_system
    git_repo --> terraform_system
    
    %% Apply styles
    class iac_hub iac
```

## Monitoring & Alert System

```mermaid
graph TB
    %% Style definitions
    classDef monitoring fill:#d1fae5,stroke:#047857,stroke-width:2px
    classDef alerting fill:#fee2e2,stroke:#991b1b,stroke-width:2px
    classDef exporter fill:#f0f9ff,stroke:#0369a1,stroke-width:1.5px
    classDef external fill:#fef3c7,stroke:#d97706,stroke-width:1.5px
    classDef target fill:#e2e8f0,stroke:#334155,stroke-width:1px
    
    subgraph caspar_monitoring["caspar - Monitoring Hub"]
        direction TB
        
        subgraph collection["メトリクス収集"]
            prometheus["Prometheus<br/>時系列DB"]:::monitoring
            uptime_kuma["Uptime Kuma<br/>外形監視"]:::monitoring
        end
        
        subgraph visualization["可視化"]
            grafana["Grafana<br/>ダッシュボード"]:::monitoring
        end
        
        subgraph alerting_system["アラート"]
            alertmanager["AlertManager<br/>アラート管理"]:::alerting
        end
    end
    
    subgraph exporters["Exporters (各サーバー)"]
        direction TB
        
        subgraph balthasar_exp["balthasar"]
            node_exp_b["Node Exporter<br/>システムメトリクス"]:::exporter
            nginx_exp_b["Nginx Exporter<br/>リクエスト統計"]:::exporter
            postgres_exp["PostgreSQL Exporter<br/>DB統計"]:::exporter
            redis_exp["Redis Exporter<br/>キャッシュ統計"]:::exporter
        end
        
        subgraph caspar_exp["caspar"]
            node_exp_c["Node Exporter"]:::exporter
            nginx_exp_c["Nginx Exporter"]:::exporter
        end
        
        subgraph other_exp["その他"]
            node_exp_nas["TrueNAS<br/>Node Exporter"]:::exporter
            node_exp_rpi["Raspberry Pi<br/>Node Exporter"]:::exporter
            node_exp_linode["Linode<br/>Node Exporter"]:::exporter
        end
    end
    
    subgraph notification["通知先"]
        direction TB
        discord_webhook["Discord Webhook<br/>運用チャンネル"]:::external
        betterstack["Better Stack<br/>外部死活監視"]:::external
    end
    
    %% Collection flows
    prometheus -->|"scrape"| node_exp_b
    prometheus -->|"scrape"| nginx_exp_b
    prometheus -->|"scrape"| postgres_exp
    prometheus -->|"scrape"| redis_exp
    prometheus -->|"scrape"| node_exp_c
    prometheus -->|"scrape"| nginx_exp_c
    prometheus -->|"scrape"| node_exp_nas
    prometheus -->|"scrape"| node_exp_rpi
    prometheus -->|"scrape"| node_exp_linode
    
    %% Visualization
    prometheus --> grafana
    uptime_kuma --> grafana
    
    %% Alerting
    prometheus --> alertmanager
    uptime_kuma --> alertmanager
    alertmanager --> discord_webhook
    
    %% External monitoring
    betterstack -->|"外部から監視"| uptime_kuma
    
    %% Apply styles
    class caspar_monitoring monitoring
```

## Storage & Backup Strategy

```mermaid
graph TB
    %% Style definitions
    classDef server fill:#e2e8f0,stroke:#334155,stroke-width:2px
    classDef beelink fill:#fef3c7,stroke:#d97706,stroke-width:2px
    classDef storage fill:#f3e8ff,stroke:#7e22ce,stroke-width:1.5px
    classDef backup fill:#dbeafe,stroke:#1d4ed8,stroke-width:1.5px
    classDef cloud fill:#dcfce7,stroke:#16a34a,stroke-width:2px
    classDef encrypted fill:#fee2e2,stroke:#dc2626,stroke-width:2px
    classDef service fill:#f8fafc,stroke:#64748b,stroke-width:1px
    classDef security fill:#fee2e2,stroke:#991b1b,stroke-width:1px
    classDef cloudflare fill:#f0fdfa,stroke:#0f766e,stroke-width:1.5px

    %% External Backup Destinations
    subgraph external_backup[クラウドバックアップ]
        r2["Cloudflare R2<br/>【DBダンプ専用】<br/>暗号化済み<br/>世代管理"]:::cloud
        filen["Filen E2EE<br/>【MinIO画像】<br/>E2E暗号化<br/>差分同期<br/>5-15分/日"]:::encrypted
    end

    %% Local Infrastructure
    subgraph local_infra[自宅インフラ]
        %% TrueNAS (Beelink ME)
        subgraph beelink_nas[Beelink ME mini - TrueNAS SCALE<br/>N100, 16GB DDR5, 2×2.5G LAN]
            emmc["eMMC 128GB<br/>TrueNAS OS"]:::storage
            
            subgraph slot23["NVMe 2TB × 2"]
                zfs_pool["ZFS Mirror Pool<br/>実効2TB<br/>圧縮有効"]:::storage
            end
            
            subgraph truenas_services[TrueNAS Services]
                backup_svc["Backup Service<br/>rsync受信<br/>rclone同期<br/>ZFS Snapshot"]:::backup
                node_exporter["Node Exporter<br/>監視エージェント"]:::service
            end
        end
        
        %% Servers
        subgraph servers["サーバー群"]
            subgraph balthasar["balthasar 本番"]
                cloudflared_bh[Cloudflared]:::cloudflare
                nginx_bh[Nginx + ModSecurity]:::security
                misskey1["Misskey"]:::service
                db1["PostgreSQL DB<br/>【本番データ】"]:::service
                minio_local["MinIO<br/>【本番データ】<br/>オブジェクトストレージ<br/>2TB"]:::storage
                backup1["Backup Agent<br/>pg_dump<br/>rsync<br/>rclone"]:::backup
            end
        end
    end

    %% Service connections - Cloudflared + Nginx経由
    misskey1 --> cloudflared_bh
    cloudflared_bh --> nginx_bh
    nginx_bh --> minio_local
    misskey1 --> db1
    
    %% DB Backup flows - 独立した2系統
    db1 -.->|"①直接外部バックアップ<br/>pg_dump + rclone<br/>世代管理<br/>TrueNAS非依存"| r2
    db1 -.->|"②ローカルバックアップ<br/>pg_dump + rsync<br/>2.5G LAN<br/>高速リストア用"| backup_svc
    backup_svc ==>|"ZFS保存<br/>スナップショット"| zfs_pool
    
    %% MinIO Backup flows - TrueNAS経由（差分管理）
    minio_local -.->|"①rsync over SSH<br/>2.5G LAN<br/>画像ファイル同期<br/>差分転送"| backup_svc
    backup_svc ==>|"②ローカル保存<br/>ZFSスナップショット"| zfs_pool
    backup_svc ==>|"③外部バックアップ<br/>rclone sync<br/>暗号化転送<br/>5-15分/日"| filen
    
    %% TrueNAS internal flows
    slot23 --> zfs_pool
    emmc --> truenas_services
    
    %% ZFS snapshots
    zfs_pool -.->|"自動スナップショット<br/>時間/日/週/月<br/>時系列バックアップ<br/>誤削除・改ざん対策"| slot23
    
    %% System backup flows
    backup1 -.->|"システムバックアップ<br/>rsync over SSH"| backup_svc
    
    %% Monitoring flows
    node_exporter -.->|"システム監視<br/>メトリクス送信"| zfs_pool
    backup_svc -.->|"バックアップ成功率<br/>転送統計<br/>暗号化検証<br/>ZFS容量監視"| node_exporter
    backup1 -.->|"R2バックアップ成功率<br/>転送統計"| node_exporter
    
    %% Apply styles
    class balthasar server
    class beelink_nas beelink
    class cloudflared_bh cloudflare
    class nginx_bh security
    class misskey1,db1,node_exporter service
    class backup_svc,backup1 backup
    class r2 cloud
    class filen encrypted
    class minio_local storage
    class slot456,slot23,zfs_pool storage
    class emmc storage
```

## Cloudflare Workers & Pages Services

```mermaid
graph TB
    %% Style definitions
    classDef workers fill:#f97316,stroke:#ea580c,stroke-width:2px,color:#ffffff
    classDef pages fill:#06b6d4,stroke:#0891b2,stroke-width:2px,color:#ffffff
    classDef github fill:#24292e,stroke:#1b1f23,stroke-width:2px,color:#ffffff
    classDef misskey fill:#86b300,stroke:#638600,stroke-width:2px,color:#ffffff
    classDef cloudflare fill:#f0fdfa,stroke:#0f766e,stroke-width:1.5px
    classDef user fill:#fef9c3,stroke:#ca8a04,stroke-width:1.5px

    %% External Services
    github_api([GitHub API<br/>Webhook/Events]):::github
    misskey_api([Misskey API<br/>yami.ski]):::misskey
    discord_api([Discord<br/>Webhook]):::github
    users([ユーザー]):::user

    subgraph cloudflare_edge["Cloudflare Edge Network"]
        direction TB

        subgraph workers_apps["Workers (サーバーレス関数)"]
            direction TB
            notifier["yamisskey-github-notifier-next<br/>GitHub開発状況 → Discord/Misskey通知"]:::workers
            yamioti["yamioti<br/>ダウン時リダイレクト"]:::workers
            signup_filter["misskey-registration-filter<br/>登録フィルター"]:::workers
            discord_notify["misskey-notify-to-discord<br/>管理者通知 → Discord転送"]:::workers
        end

        subgraph pages_apps["Pages (静的サイトホスティング)"]
            direction TB
            hub["yamisskey-hub-starlight<br/>ドキュメントサイト (Starlight)"]:::pages
            down["yamisskey-down<br/>メンテナンス・障害ページ"]:::pages
            anonote["yamisskey-anonote<br/>匿名ノートサービス"]:::pages
            revision["yamisskey-revision<br/>闘消し (ノート削除ツール)"]:::pages
            yamidao["yamidao<br/>DAO ガバナンスサイト"]:::pages
            missmap["missmap<br/>Misskeyサーバーマップ"]:::pages
        end
    end

    %% Connections - Workers
    github_api -->|"Webhook<br/>Push/PR/Issue Events"| notifier
    notifier -->|"開発状況通知"| misskey_api
    notifier -->|"開発状況通知"| discord_api

    misskey_api -->|"管理者通知"| discord_notify
    discord_notify -->|"通知転送"| discord_api

    users -->|"yami.ski<br/>アクセス"| yamioti
    yamioti -->|"ダウン時"| down
    yamioti -.->|"正常時"| misskey_api

    users -->|"新規登録<br/>リクエスト"| signup_filter
    signup_filter -->|"フィルタ済み<br/>リクエスト"| misskey_api

    %% Connections - Pages
    users -->|"ドキュメント<br/>閲覧"| hub
    users -->|"障害情報<br/>確認"| down
    users -->|"匿名ノート<br/>作成"| anonote
    users -->|"ノート<br/>削除"| revision
    users -->|"DAO参加"| yamidao
    users -->|"サーバーマップ<br/>閲覧"| missmap

    %% Apply styles
    class cloudflare_edge cloudflare
```

## Network Traffic Flow & Proxy Configuration

```mermaid
graph TB
classDef homeServer fill:#e2e8f0,stroke:#334155,stroke-width:2px
classDef service fill:#f8fafc,stroke:#64748b,stroke-width:1px
classDef security fill:#fee2e2,stroke:#991b1b,stroke-width:1px
classDef cloudflare fill:#f0fdfa,stroke:#0f766e,stroke-width:1.5px
classDef user fill:#fef9c3,stroke:#ca8a04,stroke-width:1.5px
classDef federation fill:#f3e8ff,stroke:#7c3aed,stroke-width:1.5px
classDef direct fill:#dcfce7,stroke:#16a34a,stroke-width:2px
classDef tailscale fill:#fef3c7,stroke:#d97706,stroke-width:2px
classDef storage fill:#f3e8ff,stroke:#7e22ce,stroke-width:1.5px
classDef rtc fill:#fce7f3,stroke:#db2777,stroke-width:2px

%% External actors
enduser([エンドユーザー<br/>Webブラウザ]):::user
external_servers([外部サーバー<br/>（他Misskey・画像・API等）]):::federation
bypass_services([DeepL API<br/>reCAPTCHA<br/>hCaptcha<br/>Cloudflare Challenges]):::direct

subgraph support[Support Infrastructure]
    direction TB
    
    subgraph linode[Linode Servers]
        direction TB
        subgraph proxy[linode-proxy]
            summaryproxy[Summary proxy<br/>独自IP]:::direct
            mediaproxy[Media proxy<br/>独自IP]:::direct
            squid[Squid プロキシ<br/>🔗 Tailscale ACL制限]:::tailscale
            warp[Cloudflare WARP<br/>drive.yami.ski除外]:::cloudflare
            cloudflared_p[Cloudflared]:::cloudflare
            coturn[Coturn<br/>TURN/STUN Server<br/>Synapse用<br/>UDP 3478, 5349<br/>UDP 49152-65535]:::rtc
        end
    end
    
    subgraph homeservers[🏠 自宅サーバー群]
        direction TB
        subgraph balthasar_server[balthasar - 本番環境]
            nginx_misskey[Nginx + ModSecurity<br/>WAF・Reverse Proxy]:::security
            yamisskey[Misskey<br/>🔗 Tailscale接続]:::tailscale
            minio_local[MinIO<br/>オブジェクトストレージ]:::storage
            cloudflared_bc[Cloudflared]:::cloudflare
            synapse_server[Synapse<br/>Matrix Homeserver]:::service
        end
        
        subgraph caspar_server[caspar - インフラ基盤]
            nginx_caspar[Nginx + ModSecurity]:::security
            authentik_svc[Authentik<br/>認証基盤]:::security
            mcaptcha_svc[mCaptcha<br/>CAPTCHA基盤]:::security
            cloudflared_cs[Cloudflared]:::cloudflare
        end
    end
end

%% エンドユーザーのアクセス経路（太線）
enduser ==>|"①Web UI アクセス"| cloudflared_bc
cloudflared_bc ==> nginx_misskey
nginx_misskey ==> yamisskey
nginx_misskey ==> minio_local
nginx_misskey ==> synapse_server

%% 外部サーバーからの連合リクエスト（通常線）
external_servers -->|"②連合リクエスト"| cloudflared_bc

%% MisskeyからMinIOへのCloudflared経由接続
yamisskey ==>|"③Cloudflared経由"| cloudflared_bc
cloudflared_bc ==>|"MinIOアクセス"| nginx_misskey

%% Misskeyから認証基盤へのTailscale経由接続
yamisskey -.->|"🔗 Tailscale<br/>mCaptcha認証"| mcaptcha_svc

%% Matrix VoIP (TURN経由)
synapse_server -.->|"④TURN設定<br/>turn_uris"| coturn
enduser ==>|"⑤Matrix 1:1通話<br/>STUN/TURN"| coturn

%% Misskeyサーバーからの全外部通信はSquid経由
yamisskey ==>|"⑥🔗 Tailscale経由<br/>全外部通信"| squid
squid --> warp

%% WARPからの分岐
warp -->|"外部サーバーへ"| external_servers
warp ==>|"SummaryProxy<br/>アクセス"| cloudflared_p
warp -->|"外部URL情報取得"| summaryproxy
cloudflared_p ==> mediaproxy
cloudflared_p -.-> summaryproxy
squid ==>|"MediaProxy<br/>アクセス"| cloudflared_p

%% MediaProxyからMisskeyへ画像処理結果を返却
mediaproxy ==>|"⑦画像取得/変換結果<br/>返却"| cloudflared_bc

%% SummaryProxyからの返却
summaryproxy -.->|"⑧URL情報取得結果<br/>返却"| cloudflared_bc

%% プロキシバイパス対象（特定APIサービス）
yamisskey -.->|"プロキシバイパス<br/>API直接アクセス"| bypass_services
bypass_services -.->|"API結果返却<br/>（翻訳・CAPTCHA等）"| yamisskey
```
