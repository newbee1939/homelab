# Home Environment ロードマップ

ゴール：低コストで Home Assistant を常時稼働させ、日常を楽に・楽しく。
最終的に「家のエネルギー（電気・ガス）コストとネットワーク状態の可視化／最適化」までやる。
副目的：SREとしての学びを最大化（業務で使うK8s/GitOps/Observabilityスタックを実機で運用する）。

## フェーズ全体

- **Phase 0（試用・0円）**：手持ちのWindows PCに Docker版 Home Assistant を入れて触る。気に入らなければ撤去で終わり。
- **Phase 1（本運用）**：低消費電力ホストへ移行。Linux + KVM(HAOS) + k3s + GitOps + 観測スタック。
- **Phase 2（拡張）**：電気・ガス可視化、ネットワーク可視化、必要ならノード追加でマルチノードK8s化。

ホスト選定の前提：**「電気代を可視化／最適化する」目的に対し、Windows PC常時起動はホスト消費電力で本末転倒**（30〜80W）。Phase 1 ホストは 5〜15W が必須要件。

---

## Phase 0：Windows PCで試用（今すぐ）

- [ ] Windows PCのスペック・現状を確認（OS、RAM、空き、常時電源OK か、仮想化有効化可能か）
- [ ] Docker Desktop for Windows インストール（WSL2バックエンド）
- [ ] `docker compose` で Home Assistant Container 起動（`ghcr.io/home-assistant/home-assistant:stable`）
- [ ] HA初期セットアップ＋同一LANのスマホからアクセス確認（Windowsファイアウォールで8123許可）
- [ ] 手元デバイスを1つ統合して自動化を1本動かす（SwitchBot / Tapo / Hue / Nature Remo / Chromecast 等）
- [ ] 数日〜2週間使って「続けるか／本気でやるか」判定 → Phase 1へ

---

## Phase 1：本運用（Phase 0 後に着手）

### ハード選定（Phase 0 のワークロードを見てから判断）

| 候補 | 初期費用 | アイドル消費 | 月電気代 | x86互換 | 学び |
|---|---|---|---|---|---|
| Raspberry Pi 5 (8GB) + NVMe | ~25,000円 | 5〜8W | 約150円 | ✗ ARM64 | 中（イメージ互換問題で時間取られる） |
| **中古 Lenovo ThinkCentre Tiny / HP ProDesk Mini (i5-8500T等, 16GB)** | **10,000〜18,000円** | **8〜15W** | **約200〜300円** | ✓ | **◎** |
| 新品 N100/N150 ミニPC (Beelink S12 Pro 等, 16GB/500GB) | 25,000〜32,000円 | 8〜12W | 約200円 | ✓ | ◎ |

第一候補：**中古ミニPC（ThinkCentre Tiny / OptiPlex Micro / ProDesk Mini）**。
法人リースアップが安く流れている。i5-8500T相当・16GB・SSD搭載で 1.5万円前後を狙う。

### ソフトウェア構成

```
[Mini PC]
   │
[Debian 12 (or Ubuntu 22.04 LTS)]   ← Ansible で構築自動化
   │
   ├── KVM/libvirt → Home Assistant OS (VM, 4GB / 32GB disk)   ← HAは公式VMで安定運用
   │
   └── k3s (single-node, embedded etcd)
         ├── Flux CD (GitOps)
         ├── cert-manager + Let's Encrypt (DNS-01)
         ├── Traefik / nginx-ingress
         ├── kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
         ├── Loki + Promtail (or Alloy)
         ├── sops + age（or sealed-secrets）
         ├── Tailscale operator
         └── 自作・OSS：Pi-hole, Uptime Kuma, ntopng, dashy 等
```

### 設計判断の要点

- **HAはVM、それ以外はK8s**：HA公式が想定するのは HAOS。K8s上のContainer版だとAdd-on storeが使えず学習が分散する。HAは"アプライアンス"としてVMに分離し、K8s側を運用学びの主戦場にする。
- **k3s 単ノードから**：本物のK8s API・kubectl・Helm・Manifestがそのまま動く。マルチノードはPhase 2でノード追加すれば段階移行できる。
- **GitOps (Flux)**：自宅構成をGitで管理する習慣が業務SREの基礎思考。
- **kube-prometheus-stack**：業務でほぼ確実に触るスタックを自宅で実地運用する。
- **cert-manager + DNS-01**：内部ホスト名にもパブリックLet's Encryptで証明書を発行できる、TLSの定石。

### Phase 1 タスク一覧

#### 環境構築

- [ ] ホスト選定・購入（Phase 0 後判断）
- [ ] Debian 12 / Ubuntu 22.04 LTS インストール
- [ ] SSH鍵・unattended-upgrades・UFW・fail2ban の基本ハードニング
- [ ] Ansible playbook 化（再構築できるように）
- [ ] libvirt + virt-install で HAOS VM 構築（公式 `haos_ova.qcow2` 利用）
- [ ] Phase 0 のHA configをbackupからリストア

#### Kubernetes と GitOps

- [ ] k3s インストール（single-node、Traefik無効化してnginx-ingress入れる、または Traefik のまま）
- [ ] kubeconfig取得・kubectl/k9sのローカル動作確認
- [ ] GitHub に IaC リポジトリ作成（プライベート、`infra-home` 等）
- [ ] Flux CD ブートストラップ
- [ ] sops + age でシークレット暗号化
- [ ] cert-manager + Cloudflare DNS-01 で `*.home.<your-domain>` のワイルドカード証明書

#### オブザーバビリティ

- [ ] kube-prometheus-stack を Helm で導入
- [ ] Loki + Promtail でログ集約
- [ ] node_exporter / cAdvisor / blackbox_exporter（外形監視）
- [ ] Alertmanager → 通知先（Slack / Discord / メール）
- [ ] Grafana ダッシュボード自作 1〜2枚（学習目的）

#### アクセスとバックアップ

- [ ] Tailscale で外出先からアクセス可能に
- [ ] restic で /etc /var/lib/k3s /var/lib/libvirt のスナップショット → B2 / R2
- [ ] HA backup → クラウド送信を自動化（HA Add-on or 外部スクリプト）

---

## Phase 2：可視化・最適化

### 電気代

- [ ] 電力会社にBルートサービス申込（無料、1〜2週間）
- [ ] Wi-SUN USBドングル（RL7023 Stick-D/IPS 等）購入
- [ ] HACS の `echonetlite_homeassistant` 統合
- [ ] 個別機器：SwitchBotプラグミニ or Tapo P110 を必要箇所に
- [ ] HA Energyダッシュボードで月次・時間帯別の使用量を可視化
- [ ] ピークシフト・ベース消費の発見・改善アクション

### ガス代

- [ ] 検針票の手入力 or OCR（Bルート相当の手段が乏しい）
- [ ] 季節依存の傾向把握まで

### ネットワーク可視化（J:COM環境）

- [ ] J:COM提供ルーター/HGWの型番・管理画面でできることを確認
- [ ] 方針決定：Pi-hole + ntopng同居 ／ OpenWrt挟む ／ UniFi導入
- [ ] HAダッシュボードで回線速度（Speedtest統合）と端末数モニタ

### 余裕があれば

- [ ] ノード追加してマルチノード k3s 化（Longhorn で分散ストレージ）
- [ ] PostgreSQL on K8s（CloudNativePG）
- [ ] Authentik / Authelia で SSO

---

## カバーされるSRE学習領域

✅ Linux ✅ K8s実機運用 ✅ Helm/Kustomize ✅ GitOps ✅ Prometheus/Grafana/Alertmanager
✅ Loki/PromTail ✅ TLS/cert-manager ✅ Ingress ✅ 内部DNS ✅ Backup/DR (restic→B2/R2)
✅ IaC (Ansible) ✅ Secrets管理 (sops) ✅ VPN (Tailscale/WireGuard) ✅ ログ集約

Phase 2 で追加：分散ストレージ（Longhorn）、HA Postgres（CNPG）、SSO（Authentik）

---

## ランニングコスト見込み

- ハード初期：15,000〜30,000円（一括）
- 電気代：約250円/月
- クラウド（B2バックアップ ~$1/月、独自ドメイン ~1,500円/年）：合計 ~3,000円/年
- **月およそ500円**。電気代可視化で月500円以上節約できれば実質ペイ。

---

## 参考リンク

- Home Assistant Container（Docker）: https://www.home-assistant.io/installation/windows/
- HACS: https://hacs.xyz/
- echonetlite_homeassistant: https://github.com/scottyphillips/echonetlite_homeassistant
- k3s: https://k3s.io/
- Flux CD: https://fluxcd.io/
- kube-prometheus-stack: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
