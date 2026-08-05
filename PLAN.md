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
6. **家族が使い始めた瞬間に本番になる** — 実験は実験用 VM でやる。本番を壊すと、技術ではなく信頼を失う
7. **動いたら終わりにしない** — 動いた仕組みを 1 段下まで説明できるか自問する。できないなら「道場」に回す（→ 末尾）

## 全体像

| Phase | ハード | 主なテーマ | 期間の目安 |
|---|---|---|---|
| 0 | 手持ちの Windows PC | Linux VM を建てて SSH で入る | 1 週 |
| 1 | 同上 | Ansible ですべてコード化、Docker で初サービス | 2〜4 週 |
| 2 | ミニ PC を 1 台購入 | Proxmox + Terraform、監視、24/365 稼働へ移行 | 1〜2 か月 |
| 3 | 同上 + Zigbee ドングル | Home Assistant 本格導入 | 継続 |
| 4 | Raspberry Pi / NAS など | ネットワーク・ストレージ・脱クラウド | 気が向いたら |
| 5 | スマートメーター / 太陽光 | 電気・ガス代を下げる | データが溜まってから |

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
7. **名前でアクセスできるようにする** — 内部ドメイン（`*.lab.home` など）を Pi-hole の DNS に登録し、**Caddy** をリバースプロキシに置く
   - Let's Encrypt の **DNS-01 チャレンジ**なら、外部公開せずに内部サービスも HTTPS にできる（ドメインは必要）
   - 「IP とポート番号を覚える」状態から抜けると、生活の質が一段上がる
8. **監視を入れる**（SRE の本業。ここは手を抜かない）
   - **Prometheus + Grafana + node_exporter** … CPU / メモリ / ディスク / 温度 / 電力を時系列で見る
   - `smartctl_exporter` で SSD の書き込み量と健康状態を見る（**壊れる前に気づく**ための唯一の手段）
   - **ntfy** でアラートをスマホへ。Uptime Kuma と役割を分ける（Kuma = 外形、Prometheus = 内部指標）
9. **Renovate** を GitHub に入れ、コンテナイメージ・Terraform プロバイダの更新を PR で受ける

### Done の条件

- `terraform apply` → `ansible-playbook` で、まっさらな Proxmox から全サービスが復活する
- ミニ PC の電源を抜いて挿し直しても、全部自動で戻ってくる

### ここで学ぶこと

- ベアメタル / KVM / LXC の違い（VM とコンテナは何を共有しているか）
- cloud-init による OS 初期化
- Terraform の state 管理（最初はローカルで十分。壊して学ぶ）
- TLS 証明書の中身と DNS-01 チャレンジ（なぜ外部公開なしで証明書が取れるのか）
- メトリクスの型（Counter / Gauge / Histogram）と PromQL
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
6. **本番と実験を分ける** — HA / Pi-hole / 写真は「本番」。新しい遊びは別 VM で試し、動いてから本番へ入れる
   - 家族が「電気がつかない」と言い出したら、それは**インシデント**。原則 6 を守る

### Done の条件

- HA VM を消して、バックアップから戻せた
- 自動化が最低 1 つ、毎日勝手に動いている
- 自分以外の家族が 1 つでも日常的に使っている

### 育て方（急がない）

- センサーは **1 種類ずつ**増やす。増やしたら 1 週間放置して様子を見る
- ダッシュボードは実用から作る（在室・温湿度・電力）。電力の可視化は最優先 → Phase 5 の判断材料になる
- 遊びの入口には [**Team Tracker**](https://github.com/vasqued2/ha-teamtracker)（HACS のカスタム統合。ESPN API から NBA / NFL / サッカー等のスコアをセンサー化。[専用カード](https://github.com/vasqued2/ha-teamtracker-card)もある）
- **HACS**（コミュニティ製の統合・カードを入れる仕組み）を入れると世界が広がるが、**公式統合で足りないか先に確認する**。壊れやすいのはいつも野良側

---

## Phase 4 — 広げる（必要になってから）

やりたくなった順に。全部やる必要はない。

- **脱 iCloud / 自前クラウド** → 一番元が取れる。**Immich**（写真・iOS アプリあり、iCloud 写真の代替）＋ **Nextcloud**（ファイル同期）。ここで初めて RAID / ZFS / SMART / 3-2-1 バックアップを学ぶ
  - ⚠️ 家族の写真を預ける = 消したら終わり。**バックアップが動く確証を得てから** iCloud を解約する
- **NAS** → 上を動かすなら必須。中古ミニ PC + 外付け HDD で始め、足りなくなったら 4 ベイ NAS（自作 or Synology）
- **Raspberry Pi 5** を買い足す → GPIO / **電子工作**、Pi-hole の冗長化（2 台目の DNS）、物理層の実験台
- **ネットワーク強化** → VLAN で IoT 機器を隔離、OPNsense をルータ化
- **おうち Kubernetes（k3s）** → **最後でいい**。Docker Compose で困ってから手を出す。Pi クラスタを組むのは「k8s を学ぶため」の教材としては優秀だが、**サービスを安定させる手段としては遠回り**。目的を取り違えない
- **UPS** → 停電で ZFS を壊す前に。NUT で HA から監視できる

### 生活が良くなる側（技術的には簡単。効果は大きい）

- **Jellyfin** — 手持ちの動画・音楽を全端末で。完全無料（Plex と違い機能課金なし）
- **ローカル音声アシスタント** — HA の Assist に Whisper（音声認識）+ Piper（合成）を載せる。**クラウドに声を送らずに**「電気消して」が通る。低スペックでも日本語は動く
- **Frigate**（玄関カメラ + 物体検知）— 便利だが CPU / 電力を食う。ミニ PC 1 台では重い。**やるなら最後**
- **ntfy** — バックアップ成功 / 洗濯終了 / 宅配到着をスマホへ。地味に一番使う

---

## Phase 5 — 電気・ガス代を下げる（データが溜まってから）

**目的**: 「なんとなく高い」を「どこで何 kWh 使っているか」に変える。**投資判断はその後**。

### 順番を守る（ここが要）

1. **測る** — スマートメーターの **B ルート**（電力会社に申請、無料）を HA に取り込み、家全体の消費電力を 30 分粒度で記録する
   - Wi-SUN USB ドングル（RS-WFWATTCH / BP35A1 系, 5,000〜10,000 円）が必要
   - ガスはスマートメーターが未普及なので、当面は検針票を手入力でよい
2. **貯める** — 最低 **1 年分**。夏冬の両方を見ないと判断を誤る
3. **削る（無料）** — 待機電力、給湯・エアコンの運転パターン、料金プランの見直し。**ここで数千円/月は落ちることが多い**
4. **削る（自動化）** — HA で「不在なら切る」「安い時間帯に沸かす」を組む
5. **投資を検討する** — 上をやり尽くしてから、太陽光パネル / 蓄電池 / 自家発電を**回収年数で**評価する

### 判断の目安

- 太陽光・蓄電池は初期費用 100 万円規模。**回収は 10 年前後**で、削減額のデータがないと計算できない
- 小さく試すなら**ベランダ発電**（400W 級パネル + ポータブル電源, 5〜10 万円）。工事不要で、発電の物理を体で学べる
- 賃貸なら固定設置はできない。ベランダ発電か、プラン最適化に振る

### ここで学ぶこと

- 電力量 (kWh) と電力 (W) の違い、力率、待機電力の実測
- ECHONET Lite / Wi-SUN（920MHz 帯の省電力無線）というプロトコル層
- 時系列データの扱い（HA の Recorder / InfluxDB、長期保存の設計）

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

ドメインは Phase 2 の内部 HTTPS（DNS-01）で欲しくなる。**年 1,500 円程度**で、これだけは払う価値がある。
不要なら HTTP のまま or 自己署名でも進められる。外部公開（Cloudflare Tunnel）は最後まで不要 — Tailscale で足りる。

**回収できる支出**（Phase 4-5、やるなら）

| 項目 | 費用 | 回収 |
|---|---|---|
| Immich で iCloud 200GB を解約 | HDD 代のみ | 月 400 円 × 12 = 年 4,800 円。**2 年で回収** |
| Wi-SUN ドングル（B ルート） | 5,000〜10,000 円 | 可視化による削減で数か月 |
| 太陽光 + 蓄電池 | 100 万円〜 | 10 年前後。**データを取ってから判断** |

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
| ハードウェア | Phase 2-5 | TDP と実消費電力, メモリ規格(SODIMM/DDR4/5), NVMe と SATA, 冷却 |
| 電気・電子 | Phase 4-5 | 電圧/電流/電力, GPIO, I2C, はんだ付け, 直流と交流 |

深掘りしたくなったら `deep-dive` skill を使って一次情報まで降りる。

---

## 道場（意図的に壊して直す）

**この PLAN の弱点は「うまくいくと何も学べない」こと。** 構築は最終的にドキュメント通りに動いてしまう。低レイヤーが身につくのは、**壊れた瞬間**だけ。

だから月に 1 回、実験用 VM で 1 つ選んで壊す。所要 30 分〜。**直せなくてもいい。何が起きたかを説明できれば合格。**

| # | 壊すこと | 学べること |
|---|---|---|
| 1 | `/etc/fstab` に存在しないディスクを書いて再起動 | 起動シーケンス、emergency shell、initramfs |
| 2 | ディスクを 100% 埋める（`fallocate`） | inode と block の違い、ログが書けない systemd の挙動 |
| 3 | メモリを食い潰すプロセスを動かす | OOM Killer、`exit 137`、cgroup のメモリ制限 |
| 4 | デフォルトゲートウェイを消す | ルーティングテーブル、`ip route`、DNS と疎通の切り分け |
| 5 | Docker のポートを別サービスと衝突させる | bind、`ss -tulpn`、iptables の NAT テーブル |
| 6 | ディスクを引き抜く（実験用のみ） | ファイルシステムの一貫性、fsck、SMART |
| 7 | 証明書の有効期限を切らす（時計を進める） | TLS 検証、chain of trust、NTP の重要さ |
| 8 | **バックアップから全部復元する** | **これが本命。年 1 回は必ずやる** |

やり方は毎回同じ:

1. 壊す前に「何が起きると予想するか」を `docs/` に 1 行書く
2. 壊す
3. **ログとエラーコードだけを見て**原因を推測する（すぐ検索しない）
4. 一次情報（man / 公式ドキュメント / ソース）で答え合わせをする
5. 予想が外れた箇所だけを 3 行にまとめて `docs/` に残す

### 年 1 回の「リストア訓練」

ミニ PC のディスクを消し、**バックアップと Git だけ**から全復旧する。所要時間を測って記録する。
これができれば「壊しても直せるインフラ」を持っていると言える。できなければ、それはまだ homelab ではなく**運任せの本番環境**。

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
- [Immich（公式）](https://immich.app/) — iCloud 写真の代替
- [k3s（公式）](https://docs.k3s.io/) — Phase 4 で使うなら
- [Caddy（公式）](https://caddyserver.com/docs/) — リバースプロキシ / 自動 HTTPS
- [Prometheus 公式ドキュメント](https://prometheus.io/docs/introduction/overview/) / [node_exporter](https://github.com/prometheus/node_exporter)
- [ha-teamtracker（HACS カスタム統合）](https://github.com/vasqued2/ha-teamtracker)

### 先人の記録（読み物）

- [自宅サーバーに入門してみた（Zenn）](https://zenn.dev/sonicmoov/articles/246c02cb2857d0)
- [再:自宅サーバーを始めてみよう!（YouTube, 2020）](https://www.youtube.com/watch?v=XC904l9TV6Y)
