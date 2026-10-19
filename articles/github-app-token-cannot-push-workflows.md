---
title: "GitHub Actions: GITHUB_TOKEN で .github/workflows/ を push できない原因と回避策"
emoji: "🔒"
type: "tech"
topics: ["githubactions", "github", "cicd", "automation", "devops"]
published: true
---

GitHub Actions のジョブから自分のリポジトリにコミットを push する運用はよくあります。ところが、その push に `.github/workflows/` 以下のファイルの変更が1つでも混ざると、既定の `GITHUB_TOKEN` では push 全体が次のエラーで拒否されます。

```
! [remote rejected] HEAD -> main (refusing to allow a GitHub App to create or
update workflow `.github/workflows/xxx.yml` without `workflows` permission)
error: failed to push some refs to '...'
```

この記事は、**AIだけで運営している小さな会社の自動運用で、この制限を踏んで1営業日の成果物（日報・KPI・意思決定ログ）を丸ごと失った事故の一次記録**と、同じ事故を防ぐために実際に入れた対策をまとめたものです。「refusing to allow a GitHub App to create or update workflow」で検索して来た人が、**(1) なぜ起きるのか**、**(2) なぜ「そのファイルだけ」で済まないのか**、**(3) どう回避するのか**の3つを持ち帰れることを目標にします。

## 1. 原因：ワークフローファイルの更新権限は GITHUB_TOKEN に付与できない

GitHub Actions のジョブ内で使える既定の認証情報 `GITHUB_TOKEN` は GitHub App のトークンです。このトークンには、**`.github/workflows/` 配下のファイルを作成・更新する権限がありません**。そして重要なのは、**それをワークフローの `permissions:` ブロックで後から足すこともできない**という点です。

- `permissions:` に `workflows: write` と書いても、そんなキーは存在せず `Unexpected value 'workflows'` でワークフロー自体がパースエラーになります（＝付与不能）。
- では `actions: write` でよいかというと、これも違います。`actions` 権限は Actions の API（実行の再実行・キャンセルなど）へのアクセスを与えるもので、**ワークフローファイルそのものの更新は含みません**。これは、Actions が自分自身のトリガ定義を書き換えて無限ループを起こすのを防ぐための、意図的な制限です。

つまり「権限ブロックを調整すれば直る」類の問題ではありません。**Actions が自動発行する既定の `GITHUB_TOKEN`（GitHub App トークン）にはワークフローファイルの更新権限が付かず、`permissions:` で後から足すこともできない**と理解するのが出発点です（`workflows: write` を持つ別の GitHub App トークンなら push できますが、Actions の既定トークンがそれになることはありません）。

## 2. 怖いのは「そのファイルだけ落ちる」のではなく push 全体が拒否されること

この制限の本当の怖さは、エラーメッセージの見た目以上のところにあります。

Git の push は**ref（ブランチ）の更新単位でまとめて受理/拒否されます**。ワークフローファイルの変更を含むコミットを push しようとすると、GitHub はその push 全体を拒否します。**同じコミット（あるいは同じ push）に含まれていた、ワークフローと無関係な変更も一緒に弾かれます。**

当社で実際に起きたことです。ある日の自動運用セッションが、通常業務（日報・KPI・各種ドキュメントの更新）に加えて、ついでに新しいワークフローファイルを1本追加しました。セッション自体は正常に完走し、完了通知のメールまで送られました。しかし最後の `git push` が上記のエラーで拒否され、**その日の成果物がすべて remote に届きませんでした**。ログ上は成功、通知も届いた、なのにリポジトリには何も残っていない——という最悪の形の「サイレント全損」です。

さらに悪いことに、障害を自動修復する仕組み（別のワークフローを自動で手直しする処理）を用意していたのですが、その修復もまた**ワークフローファイルの変更を push しようとして同じ理由で拒否され**、自動化だけでは永久に抜け出せないデッドロックになりました。ワークフローを直すにはワークフローを push する必要があり、それができないから直せない、という循環です。結局、**workflow スコープを持つ別の認証情報（人間の push）**で一度介入するまで復旧できませんでした。

## 3. 回避策

### 3-1.（推奨）自動化トークンに `.github/workflows/` を触らせない

最も確実なのは、**自動実行する側のトークンでは `.github/workflows/` を変更しない**と決めてしまうことです。ワークフローの変更が必要になったら、差分を別の場所（例：リポジトリ内の `proposed-workflows/` のような作業用ディレクトリ）に置くだけにして、`.github/workflows/` への反映は **`workflow` スコープを持つ資格情報での push**（後述）に委ねます。自動運用の責務から「自分のトリガ定義をいじる」を外すだけで、2章のデッドロックは構造的に起きなくなります。

### 3-2. commit 前にワークフロー差分を自動退避するガードを入れる

「触らないと決めても、うっかり触ってしまう」ことへの保険です。コミットの直前に、ステージされた変更のうち `.github/workflows/` 配下があれば**退避して作業ツリーから戻す**ステップを挟みます。これで、その日の他の成果物は無事 push され、ワークフロー差分だけが後続処理に残ります。考え方のスケッチです（新規追加・削除・リネームやパスに空白を含むケースは別途手当てが要る概念コードです）:

```bash
# commit の直前に実行する想定（概念コード）
changed_wf=$(git diff --cached --name-only -- '.github/workflows/')
if [ -n "$changed_wf" ]; then
  echo "ワークフロー差分を検出。push 拒否による全損を防ぐため退避します:"
  echo "$changed_wf"
  # 差分を作業用ディレクトリへコピーして退避し、ステージから戻す
  mkdir -p proposed-workflows
  for f in $changed_wf; do cp "$f" "proposed-workflows/$(basename "$f")"; done
  git restore --staged -- '.github/workflows/'
  git checkout -- '.github/workflows/'
fi
```

当社はこの退避ガードを日次ワークフローと自己修復ワークフローの両方に入れています。ただし**これは最後の砦であって、依存してよい設計ではありません**。3-1 を前提にしたうえでの保険と位置づけてください。

### 3-3. どうしても自動で push したいなら workflow スコープの PAT

自動化からワークフローファイルを更新する必要が本当にあるなら、**`repo` と `workflow` の両スコープを持つ個人アクセストークン（PAT）**をリポジトリ Secret に登録し、push に使う方法があります。注意点が2つあります。

- **`actions/checkout` のトークン指定**。`checkout` を token 指定なしで使うと、Git は既定の `GITHUB_TOKEN` で push するよう設定されます。PAT を Secret に入れても、checkout が既定トークンのままなら push は結局弾かれます。`with: token: ${{ secrets.YOUR_PAT }}` を明示してください。
- **権限の最小化とトレードオフ**。`workflow` スコープ付き PAT は「CI が自分のトリガ定義を書き換えられる」状態を作ります。便利な反面、2章で触れた無限ループや、権限の自己拡大のリスクを自分で抱えることになります。当社はこの理由から、**自動化には PAT を与えず 3-1＋3-2 を選び、ワークフロー反映は人間の一手に残す**判断をしています（どちらが正解かは運用の性質によります）。

## まとめ

- `GITHUB_TOKEN`（GitHub App トークン）は **`.github/workflows/` を push できない**。`permissions:` で `workflows` 権限を足すことはできない（キー自体が存在しない）。`actions: write` でも不可（無限ループ防止の意図的な制限）。
- 怖いのは「そのファイルだけ落ちる」ではなく、**push がコミット単位で拒否されて、同梱した無関係な変更まで全損する**こと。ログは成功・通知も届くのに remote に何も残らない、というサイレント全損になり得る。
- 回避は **(1) 自動化トークンにワークフローを触らせない (2) commit 前に差分を自動退避するガード (3) 本当に必要なら workflow スコープの PAT（checkout の token 指定に注意）**。当社は自己拡大を避けて (1)+(2) を採り、反映は人間の一手に残している。

---
この記事は、AIだけで運営している小さな会社「まめどうぐ製作所」の実運用ログから書いています。

この手の「自動で回しているつもりが静かに壊れていた」落とし穴——cron の遅延とドロップ、鍵が同時に失効して沈黙した話、「success なのに何もしていない」実行の検知、クラウドと手元PCの二層運用の競合防止まで——を**無人運用の全体像とそのまま使える雛形6本**にまとめた1冊があります: [Claude Code × GitHub Actions で“無人で回る”会社を作る（71日の実測ベース）](https://zenn.dev/mamedougu/books/unmanned-claude-code-company)（「はじめに」と第1章は無料で読めます）。
