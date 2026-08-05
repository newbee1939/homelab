# HomeLab PLAN

盆栽のように、小さく作って長く育てる。

## ゴール

- 自宅に「壊しても直せるインフラ」を持ち、Linux / OS / ネットワーク / ハードウェアを手を動かして学ぶ
- 最終的に Home Assistant を主役に据える
- **すべての設定をコードで再現できる**（手作業の設定を残さない）
- 運用費は電気代のみを目指す（ソフトは無料枠で完結させる）

## 原則

1. **手で設定したら負け** — 一度手で試してよい。動いたら必ず Ansible / Terraform に落として、消して作り直す
2. **1 フェーズ 1 台** — ハードを増やすのは、今の台数で困ってからにする
3. **ポートは開けない** — 外からのアクセスは Tailscale。ルータのポート開放は原則しない
4. **壊す前提** — いつでも OS を入れ直せる状態を保つ。データだけをバックアップする
5. **学びを残す** — 詰まった箇所は `docs/` に 1 ファイル書く。それが homelab の資産

## 全体像

| Phase | ハード | 主なテーマ | 期間の目安 |
|---|---|---|---|
| 0 | 手持ちの Windows PC | Linux VM を建てて SSH で入る | 1 週 |
| 1 | 同上 | Ansible ですべてコード化、Docker で初サービス | 2〜4 週 |
| 2 | ミニ PC を 1 台購入 | Proxmox + Terraform、24/365 稼働へ移行 | 1〜2 か月 |
| 3 | 同上 + Zigbee ドングル | Home Assistant 本格導入 | 継続 |
| 4 | Raspberry Pi / NAS など | ネットワーク・ストレージ・冗長化 | 気が向いたら |

---

## Phase 0 — Windows PC で Linux を建てる（無料・買い物ゼロ）

**目的**: 「LAN 上に IP を持った Linux サーバー」を 1 台手に入れる。

### やること

1. Windows のエディションを確認する（`winver`）
   - **Pro / Enterprise** → Hyper-V を有効化（推奨。仮想スイッチを「外部」にすると VM が LAN 上に自分の IP を持つ）
   - **Home** → Hyper-V が使えないので VirtualBox で代用（ネットワークは「ブリッジアダプター」）
2. Ubuntu Server LTS を VM にインストール（GUI なし。CPU 2 / RAM 4GB / Disk 32GB で十分）
3. ルータの DHCP で VM の MAC アドレスに固定 IP を予約する
4. Windows から SSH 鍵でログインできるようにする（パスワード認証は切る）

### Done の条件

- `ssh homelab` で鍵だけで入れる
- VM を一度消して、同じ手順で作り直せた

### ここで学ぶこと

- 仮想化とは何か（ハイパーバイザ Type-1 / Type-2 の違い）
- NAT とブリッジの違い＝「ルータから見て VM が居るか居ないか」
- SSH 公開鍵認証、`systemd` の起動シーケンス（`systemd-analyze blame`）

---

## Phase 1 — すべてを Ansible に落とす（無料・買い物ゼロ）

**目的**: Phase 0 で手で入れた設定を、**コマンド 1 発で再現できる状態**にする。

### やること

1. `ansible/` を作り、Phase 0 の手作業を playbook 化する
   - ユーザー / SSH 設定 / `ufw` / タイムゾーン / 自動更新（`unattended-upgrades`）
2. Docker を Ansible で入れ、最初のサービスを Docker Compose で立てる
   - **Pi-hole**（DNS 広告ブロック）… 効果が体感でき、DNS を学べる
   - **Uptime Kuma**（死活監視）… 自分のラボを自分で監視する
3. **Tailscale** を入れ、外出先から自宅の Pi-hole 管理画面に入れるようにする（Personal プランは無料）
4. GitHub Actions で `ansible-lint` / `--check`（dry-run）を回す

### Done の条件

- VM を作り直し → `ansible-playbook site.yml` だけで Phase 1 の状態に戻せる
- 家の全端末の DNS が Pi-hole を向き、広告が消えている

### ここで学ぶこと

- 冪等性（何回流しても同じ結果になる、が Ansible の本質）
- DNS の解決順序、`/etc/resolv.conf`、`dig` の読み方
- Docker のネットワーク / ボリューム、`iptables` に何が書かれるか
- WireGuard ベースの VPN が NAT 越えをどうやるか（Tailscale の中身）

---

## Phase 2 — ミニ PC を買って Proxmox へ（ここで初めてお金を使う）

**目的**: Windows PC を解放し、24 時間動く本番機を持つ。仮想マシンを Terraform で作る。

### やること

1. ミニ PC を 1 台買う（→ 下の「ハードウェア選定」）
2. **Proxmox VE 9.x** をインストール（Debian 13 ベースの Type-1 ハイパーバイザ、無料）
   - Enterprise リポジトリを無効化し、No-Subscription リポジトリに切り替える
3. Proxmox に API トークンを作り、**Terraform（`bpg/proxmox` プロバイダ）** で VM / LXC を定義する
   - `telmate/proxmox` は更新が止まりがち。今は `bpg/proxmox` が事実上の標準
4. Terraform で VM を作る → Ansible で中身を作る、の 2 段構えにする
5. Phase 1 のサービスを新ホストへ移す。Windows PC の VM は捨てる
6. バックアップを決める（Proxmox Backup で VM を、`restic` で設定・データを外付け HDD へ）

### Done の条件

- `terraform apply` → `ansible-playbook` で、まっさらな Proxmox から全サービスが復活する
- ミニ PC の電源を抜いて挿し直しても、全部自動で戻ってくる

### ここで学ぶこと

- ベアメタル / KVM / LXC の違い（VM とコンテナは何を共有しているか）
- cloud-init による OS 初期化
- Terraform の state 管理（最初はローカルで十分。壊して学ぶ）
- 実測の消費電力（ワットチェッカーを 1 個買うと世界が変わる）

---

## Phase 3 — Home Assistant を育てる

**目的**: 家が便利になる。ここからが盆栽。

### やること

1. **Home Assistant OS を VM として** Proxmox に建てる（公式にサポートされる形態。Supervisor とアドオンストアが使える）
2. まず既存の家電・Wi-Fi 機器を統合する（買い物ゼロで始められる）
3. **Zigbee USB ドングル**（SLZB-06 / SkyConnect 系, 3,000〜6,000 円）を追加し、センサーを 1 個だけ買って動かす
   - 温湿度センサー → 「暑くなったらエアコン」など、成功体験を 1 つ作る
4. 設定を Git 管理する（`configuration.yaml` と自動化。Secrets は分離）
5. 自動バックアップ（HA のバックアップ → NAS / 外付けへ）

### Done の条件

- HA VM を消して、バックアップから戻せた
- 自動化が最低 1 つ、毎日勝手に動いている

### 育て方（急がない）

- センサーは **1 種類ずつ**増やす。増やしたら 1 週間放置して様子を見る
- 電気・ガスのスマートメーター連携（B ルート / HEMS）は TODO の「電気代を減らす」への入口
- 太陽光・蓄電池は**まず HA で消費電力を可視化してから**判断する（データなしに買わない）

---

## Phase 4 — 広げる（必要になってから）

やりたくなった順に。全部やる必要はない。

- **Raspberry Pi 5** を買い足す → GPIO / 電子工作、Pi-hole の冗長化、物理層の実験台
- **NAS / 自前クラウド** → iCloud をやめる（Immich で写真、Nextcloud でファイル）。ここで初めて RAID / ZFS / SMART を学ぶ
- **ネットワーク強化** → VLAN で IoT 機器を隔離、OPNsense をルータ化
- **Kubernetes（k3s）** → 最後でいい。Docker Compose で困ってから。「おうち k8s」は目的ではなく手段
- **UPS** → 停電で ZFS を壊す前に。NUT で HA から監視できる

---

## ハードウェア選定（2026 年 7 月時点）

### 結論

**中古 1L ミニ PC（ThinkCentre Tiny 系）が第一候補。予算最優先なら新品 N150 ミニ PC。**

| 候補 | 価格の目安 | アイドル電力 | 向き / 不向き |
|---|---|---|---|
| **中古 ThinkCentre M75q-1 / M720q / M920q Tiny** | 20,000〜25,000 円 | 8〜12W | ◎ RAM 64GB まで載る / 分解・増設しやすく**ハードの勉強に最適** / 企業向けで堅牢。△ 個体差あり |
| **新品 N150 ミニ PC**（GMKtec G3 Plus, BMAX B1 Pro など） | 18,000〜25,000 円 | 7〜8W | ◎ 最安・最省電力・保証あり / RAM 16GB・SSD 込み。△ RAM 上限が低く拡張性が薄い |
| **Minisforum MS-01 等の上位機** | 80,000 円〜 | 20W〜 | ◎ 10GbE・PCIe。今は不要。**買わない** |
| **Raspberry Pi 5** | 15,000 円〜（周辺込み） | 4〜7W | ◎ GPIO / 電子工作。△ ARM・I/O が弱くサーバー本命には不向き。**Phase 4 で買う** |

### 判断基準（迷ったらこれ）

- **RAM は 16GB 以上、できれば増設できる機種**。CPU よりメモリで詰まる
- **有線 LAN 必須**（できれば 2.5GbE）。Wi-Fi でサーバーを運用しない
- **NVMe スロット + 2.5inch ベイ**があると後で困らない
- N100 と N150 は実質同性能。**安い方を買う**

### 買わなくていいもの

ラック、10GbE、サーバー用 CPU、UPS（Phase 4 まで）、複数台クラスタ。全部「困ってから」。

---

## コスト

| 項目 | 費用 |
|---|---|
| Proxmox VE / Ansible / Terraform / Docker / Home Assistant | 0 円 |
| Tailscale Personal | 0 円 |
| GitHub（private repo / Actions） | 0 円 |
| **電気代**（8W 24h・31 円/kWh 換算） | **約 180 円/月** |
| 初期投資（ミニ PC） | 20,000 円前後 |
| Zigbee ドングル + センサー 1 個 | 5,000 円前後 |

ドメインを取って Cloudflare Tunnel で公開したくなったら年 1,500 円程度。**Phase 3 までは不要**（Tailscale で足りる）。

---

## リポジトリ構成

最初はこれだけ。増やさない。

```
homelab/
├── PLAN.md
├── ansible/
│   ├── inventory.yml
│   ├── site.yml
│   └── roles/
├── compose/          # サービスごとの docker-compose.yml
├── terraform/        # Phase 2 から
└── docs/             # 学習ログ・詰まった記録
```

- Secrets は Ansible Vault か SOPS。**平文を Git に置かない**
- `docs/` は綺麗に書かない。「何が起きて、何が原因で、どう直したか」の 3 行でいい

---

## 低レイヤー学習トラック

フェーズを進めると自然に触るテーマ。**先に本を読まない。詰まってから該当箇所を読む。**

| 層 | いつ | 触るもの |
|---|---|---|
| プロセス / OS | Phase 0-1 | `systemd`, プロセスの親子, シグナル, exit code, `strace` |
| ファイルシステム | Phase 0-2 | パーティション, LVM, マウント, inode, `df` と `du` の差 |
| ネットワーク | Phase 1-4 | ARP, DNS, DHCP, NAT, `tcpdump`, ルーティングテーブル, VLAN |
| 仮想化 | Phase 2 | KVM, namespace / cgroup, virtio, ブリッジ |
| ストレージ | Phase 4 | ZFS, RAID, SMART, 書き込み耐性(TBW) |
| ハードウェア | Phase 2-4 | TDP と実消費電力, メモリ規格(SODIMM/DDR4/5), NVMe と SATA, 冷却 |

深掘りしたくなったら `deep-dive` skill を使って一次情報まで降りる。

---

## 最初の一歩（今日やること）

1. `winver` で Windows のエディションを確認する
2. Pro なら Hyper-V を有効化、Home なら VirtualBox を入れる
3. Ubuntu Server LTS の ISO を落とす

ここまでで Phase 0 の半分が終わる。

---

## 参考

- [Proxmox VE 9.0 リリース（公式）](https://proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-0)
- [bpg/terraform-provider-proxmox（公式リポジトリ）](https://github.com/bpg/terraform-provider-proxmox) / [Terraform Registry](https://registry.terraform.io/providers/bpg/proxmox/latest)
- [Home Assistant インストール方法（公式）](https://www.home-assistant.io/installation/)
- [Tailscale 無料プラン（公式ドキュメント）](https://tailscale.com/docs/account/manage-plans/free-plans-discounts)
- [Ansible ベストプラクティス（公式）](https://docs.ansible.com/ansible/latest/tips_tricks/index.html)
