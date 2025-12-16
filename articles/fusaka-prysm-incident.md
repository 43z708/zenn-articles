---
title: "【1.8億円の報酬損失】Ethereum Fusaka直後のPrysm障害に学ぶ、「安全な」冗長化構成"
emoji: "🔖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ethereum", "prysm", "web3signer", "ブロックチェーン", "Fusaka"]
published: true
publication_name: omakase
published_at: 2025-12-16 15:00
---

## はじめに

Fusaka 実装直後に発生した Prysm の障害は、私たちノード運用者に「単一クライアント依存のリスク」と「誤った冗長化による Slashing の恐怖」を同時に突きつける出来事でした。

[Prysm Postmortems](https://prysm.offchainlabs.com/docs/misc/mainnet-postmortems/) の事後分析によると、Prysm 側のバグによって参加率が最大 75%まで低下し、影響を受けたバリデータ群の合計で 382 ETH 相当もの報酬機会損失が発生したとされています。

この記事では、EL/CL/VC を複数構成にしつつ、署名を Web3Signer に集約して二重署名を防ぐための安全な設計について、運用者の視点で整理します。

## Prysm障害の詳細

Fusaka はメインネットで Epoch 411392（2025-12-03 21:49:11 UTC）に有効化されましたが、その直後、Prysm ノードが過去ステートを過剰に生成してしまい、計算資源を大量消費する事態に陥りました。これにより深刻な性能低下が発生し、アテステーションやブロック確定が遅延し、結果として Prysm のバリデータ参加率は一時 75%まで急落しました。

幸い、緊急回避策の適用と修正版（v7.0.1 / v7.1.0）のリリースにより、24 時間以内に参加率は 99%水準まで回復しています。また、Prysm のシェアが 18〜22%程度に留まっていたことや、他クライアントが正常に稼働し続けたことが、コンセンサス維持の救いとなりました。

## ノード管理で重要な指向は"多少停止してでも"誤署名を避ける」こと

今回の事象自体は「性能劣化」と「参加率低下」で、投票参加しないことによる機会損失でした。もちろん避けたいのですが、復旧を急ぐあまり二重署名を誘発してしまうことは、絶対に避けなければなりません。

結論として、バリデータの冗長化は次の順序で考える必要があります。

1. EL/CL を冗長化し、可能ならクライアント多様性（Multi-Client）を取り入れる
2. VC を冗長化する場合は、同一鍵が同時に署名できない設計に縛る
3. 署名を Web3Signer に集約し、Slashing Protection を一箇所に寄せる

## 前提知識：EL / CL / VC / Signer の責務分離

Ethereum のバリデータ運用において、設計の中心となるのは「どこで署名するか」です。VC（Validator Client）を増やすこと自体は簡単ですが、鍵と署名機能を分散させた瞬間に「整合性の取れない Slashing Protection」が生まれてしまいます。

そこで有効なのが、Remote Signer（EIP-3030）です。VC から秘密鍵を分離し、署名機能を外部に出す構成です。

<!-- textlint-disable -->

:::message alert
注意
Lighthouse のドキュメント等でも警告されているとおり、Remote Signer は「銀の弾丸」ではありません。構成が複雑化し、新たなセキュリティリスクと Slashing リスクが増えるため、導入は“設計と運用が追いつく範囲”に限定すべきです。
:::

<!-- textlint-enable -->

## Web3Signerが提供する「最低限の安全装置」

Web3Signer は、署名済み履歴を DB に記録し、それと矛盾するブロックやアテステーションへの署名を拒否する Slashing Protection を提供します。Slashing Protection はデフォルトで有効で、対応 DB は PostgreSQL のみです。

さらに重要なのが可用性の確保です。Web3Signer は複数インスタンスが同じ Slashing DB に接続でき、DB ロックにより「同一鍵を複数インスタンスがロードしても、実際に署名するのは 1 つだけになる」ように設計されています。つまり、Signer 自体も HA 構成に載せられます。

<!-- textlint-disable -->

:::message alert
注意
Web3Signer（EIP-3030の外部署名）を前提にした運用は、Validator Client（VC）側の対応状況と制約に依存します。
主要VCはWeb3Signer連携の手順が用意されていますが、たとえばPrysmは「VC 1つにつき Web3Signer 1つ」かつ「ローカル鍵とリモート鍵の混在不可」といった制約があります。
この構成を採用する前に、利用VCの制約（鍵移行・混在可否・切替手順）を必ず確認してください。
:::

<!-- textlint-enable -->

## Omakase推奨アーキテクチャ

以下の推奨アーキテクチャは、一定以上の規模のバリデータに限ります。小規模バリデータの場合、冗長化によるコストが収益を上回りやすいため非推奨です。

![アーキテクチャ](/images/fusaka-prysm-incident/architecture.png)

| コンポーネント         | 推奨方針                                                                                                                                                                      |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| EL（Execution Layer）  | 2系統以上を前提とし、異なるクライアント実装を組み合わせて冗長化します。                                                                                                       |
| CL（Consensus Layer）  | 2系統以上を前提とし、異なるクライアント実装を組み合わせて冗長化します。                                                                                                       |
| VC（Validator Client） | 複数台を用意しますが、同一バリデータ鍵を「同時に2台へアクティブ割当」しません。負荷分散したい場合は、鍵を分割してVCごとに担当を固定し、各VCに対してActive/Standbyを組みます。 |
| Signer                 | Web3Signerに集約します。Web3Signerは複数台を同一PostgreSQL（Slashing DB）へ接続し、Signer自体もHA構成にします。                                                               |

## 署名系の設計方針

Remote Signer（EIP-3030）を採用し、VC が HTTP 経由で署名を依頼する形を取ります。ここで「Slashing Protection をどこに置くか」を固定します。弊社では Web3Signer の Slashing DB を「単一の真実（Single Source of Truth）」として扱います。

Web3Signer は Slashing DB 前提で設計されており、DB ロックによって署名競合を避けられるため、Signer の冗長化が現実的になります。一方で、Remote Signer 特有の複雑さは増えるため、監視と切替手順の整備は必須です。

## まとめ

Fusaka 直後の Prysm 障害は、クライアント多様性（Client Diversity）がネットワークのレジリエンスに直結することをあらためて示しました。

一方で、個別のノード運用者にとっての最大のリスクは「障害対応中の二重署名」です。EL/CL/VC を冗長化するなら、署名を Web3Signer に集約し、Slashing Protection を「単一の真実」として管理します。さらに VC 運用は、同一鍵の同時アクティブ割当を禁止し、鍵分割＋Active/Standby で設計します。この原則を守ることが、安全なステーキング運用の要になります。

## 引用元

- Prysm Postmortems :contentReference[oaicite:22]{index=22}
- Prysm Docs: Use Web3Signer :contentReference[oaicite:23]{index=23}
- Lighthouse Book: Web3Signer :contentReference[oaicite:24]{index=24}
- Teku Docs: Use Web3Signer :contentReference[oaicite:25]{index=25}
- Nimbus Docs: External signer :contentReference[oaicite:26]{index=26}
- Lodestar Docs: External signer :contentReference[oaicite:27]{index=27}
- Consensys Web3Signer Docs: Architecture（slashing DB / DB locking）:contentReference[oaicite:28]{index=28}
- CoinPost :contentReference[oaicite:21]{index=21}
- Cointelegraph :contentReference[oaicite:29]{index=29}
- Crypto.news :contentReference[oaicite:30]{index=30}
