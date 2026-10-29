# AWX ワークフロー 体験用サンプル

ワークフローの骨格（スケルトン）を、手を動かして覚えるための最小構成です。

## 構成図

```
                 success              success               success
 [01 prepare] ─────────▶ [02 check] ─────────▶ <承認 approve> ─────────▶ [03 deploy]
                              │                     │                     │      │
                              │ failure             │ failure             │      │ success
                              │                     │ (Deny/Timeout)      │      ▼
                              ▼                     ▼                     │  [05 report]
                         [04 rollback] ◀────────────┴──────── failure ────┘      ▲
                              │                                                  │
                              └──────────────────── always ──────────────────────┘
```

| 覚えること | どこで体験するか |
|---|---|
| ① success / failure / always の3種類の線 | check の成否、rollback → report |
| ② 承認ノード（人が止める） | approve |
| ③ ノード間の値渡し（set_stats → artifacts → extra_vars） | build_id, deploy_result |
| ④ 合流（convergence） | report に2経路から入る |
| ⑤ サーベイで実行時に分岐を切り替え | force_fail |

## ファイル

```
playbooks/01_prepare.yml   build_id を作って set_stats で後続へ
playbooks/02_check.yml     force_fail=yes なら失敗（分岐の起点）
playbooks/03_deploy.yml    承認後の本作業。deploy_result=OK を後続へ
playbooks/04_rollback.yml  失敗/却下時。deploy_result=ROLLED_BACK を後続へ
playbooks/05_report.yml    合流点。受け取った値を全部表示
setup/create_awx_objects.yml  AWX上のオブジェクト一式を作る
```

## 手順

### 1. このディレクトリを Git リポジトリに置く
AWX のプロジェクトは Git から取得するため。公開リポジトリが一番楽です
（社内 Gitea/GitLab でも可。非公開ならプロジェクトに SCM 認証情報が必要）。

```bash
git init -b main && git add . && git commit -m "awx workflow sample"
git remote add origin <URL> && git push -u origin main
```

### 2. AWX にオブジェクトを作る

**A. 自動（おすすめ）**
```bash
ansible-galaxy collection install awx.awx
export CONTROLLER_HOST=https://<awx>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD='...'
export CONTROLLER_VERIFY_SSL=false      # 自己署名証明書なら
ansible-playbook setup/create_awx_objects.yml -e scm_url=<URL>
```

**B. 手作業（UIで1回は作ってみると骨格が一番身につきます）**
1. インベントリ `wf-sample-inv` → ホスト `localhost`、変数 `ansible_connection: local`
2. プロジェクト `wf-sample-project`（Git, URL指定）→ 同期が緑になるのを確認
3. ジョブテンプレート 5個（01〜05、それぞれ上記プレイブック）
4. テンプレート → 追加 → **ワークフローテンプレート** `wf-sample-workflow`
5. ビジュアライザーで START から図のとおりにノードを繋ぐ
   - 承認ノードはノード追加時の種類で「承認」を選ぶ
   - 線の種類（成功時／失敗時／常時）はノード追加時に選択
6. ワークフローにサーベイ追加：変数 `force_fail`、複数選択 `no` / `yes`、既定 `no`

### 3. 実験（ここが本題）

| # | 実行時の操作 | 期待される流れ | 確認ポイント |
|---|---|---|---|
| 1 | force_fail=`no` → 承認で **Approve** | 01→02→承認→03→05 | 05 の出力に build_id と `deploy_result=OK` |
| 2 | force_fail=`no` → 承認で **Deny** | 01→02→承認→04→05 | 承認の却下は「failure」の線に乗る |
| 3 | force_fail=`yes` | 01→02✕→04→05 | 承認ノードは**実行されない**（灰色） |
| 4 | 承認を放置（timeout 600秒） | 01→02→承認⌛→04→05 | タイムアウトも failure 扱い |

承認待ちは、画面上部のベルアイコン／「承認」一覧、またはワークフロー画面の承認ノードから Approve / Deny します。

各実験で見るところ:
- **ワークフロー出力画面**：どのノードが走り、どれが灰色（Do Not Run）になったか
- **各ジョブの「詳細」→ 追加変数**：前ノードの set_stats がここに届いている
- **05 report の出力**：`awx_workflow_job_id` が入っている（AWX が自動注入する変数）
- **ワークフロー全体のステータス**：実験3では 02 が失敗しても、失敗経路で処理されているので全体がどう表示されるか

### 4. 余裕があれば改造してみる
- report の `all_parents_must_converge` を **true** にして実験1を流す → report が走らなくなる理由を考える（rollback が走らないので親が揃わない）
- 01 の `per_host: false` を消す → 後続に build_id が届かなくなることを確認
- ノードを1つ「ワークフロー」型にしてネスト（ワークフローの中のワークフロー）

## ローカルでの動作確認（AWXなし）
プレイブック単体は普通の ansible-playbook で動きます（値渡しは -e で手動模擬）。
```bash
ansible-playbook -i localhost, -c local playbooks/02_check.yml -e build_id=X -e force_fail=yes
```
