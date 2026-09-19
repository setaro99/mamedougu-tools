---
title: "GitHub Actions で Claude Code を63日間毎晩走らせて踏んだ落とし穴5つ（実測つき）"
emoji: "🌙"
type: "tech"
topics: ["claudecode", "githubactions", "ai", "automation", "agent"]
published: false
---

<!-- 技術出版事業部 記事① ／ 独立QA合格前は published: false のまま。数字はすべて当社リポジトリの実測ログに基づく -->

小さなデジタル商品の会社を、人間が日常的に関与せず **Claude Code + GitHub Actions だけで毎晩1営業日回す** 構成を63日間運用しました。
「動かす」までは半日でしたが、「止まらずに動き続ける」には別の知識が要りました。この記事はその実測メモです。

対象: `anthropics/claude-code-action` を `schedule` で回している／回そうとしている人。

## 構成（最小）

```yaml
on:
  schedule:
    - cron: "37 11 * * *"   # 20:37 JST。理由は後述
  workflow_dispatch: {}
permissions:
  contents: write
  issues: write
jobs:
  business-day:
    runs-on: ubuntu-latest
    timeout-minutes: 110
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          claude_args: >-
            --max-turns 250
            --allowedTools "Read,Write,Edit,Glob,Grep,Bash(git:*),Bash(python3:*),Bash(gh issue:*)"
          prompt: |
            （リポジトリ内の手順書を読んで1営業日を実行する）
      - name: Commit and push results
        run: |
          git add -A && git commit -m "営業日 $(date +%F)" || exit 0
          git push
```

これで動きます。以下は、これだけでは**止まる**理由です。

## 1. `schedule` は指定時刻に走らない（実測 +3.5〜4.5時間）

`cron: "37 14 * * *"`（23:37 JST）で1か月運用した実測では、実際の開始は **03:09〜04:12 JST**。
毎日 3.5〜4.5 時間遅れ、しかも日によってぶれます。GitHub の公式ドキュメントも「高負荷時は遅延しうる」と書いていますが、体感はそれより大きい。

困るのは2点です。

- **利用枠**: Claude の5時間ウィンドウが日中の作業時間に食い込む。当社は「平日は朝9時に枠が空いている状態」を守るため、起点を **20:37 JST に前倒し**して遅延込みで 23〜01 時開始・04 時までに完了、に収めました。
- **日付**: 日をまたぐと `date +%F` が日によって当日/翌日に変わり、日報ファイル名が飛びます。当社は **ランナーの TZ を JST+4h（`TZ: Etc/GMT-13`）に固定**し、20:00〜翌07:59 JST に始まった実行はすべて「翌営業日」として扱うことで解決しました。

```yaml
env:
  TZ: Etc/GMT-13   # JST+4h。夜勤が「翌営業日分」を処理する運用
```

`00分` 起点は特に混むので避けるのも効きます。

## 2. `GITHUB_TOKEN` は `.github/workflows/` を push できない — しかも**その日の成果が全部消える**

Actions の `GITHUB_TOKEN` には `workflows` 権限が無く、ワークフローファイルを含むコミットの push は次のエラーで拒否されます。

```
refusing to allow a GitHub App to create or update workflow ... without `workflows` permission
```

問題は「そのファイルだけ落ちる」のではなく、**コミット単位で push が拒否される**こと。エージェントが「自動化を増やそう」と気を利かせて
ワークフローを1本追加した日は、日報も台帳もその日の成果が丸ごと失われました。さらに、失敗時に走る自己修復ジョブも同じ理由で自分の修正を
push できず、**構造的なデッドロック**になります。

対策は「push 前にワークフロー差分を退避する」ガードです。

```bash
wf_changed=$(git status --porcelain -- .github/workflows | awk '{print $2}')
if [ -n "$wf_changed" ]; then
  mkdir -p operations/proposed-workflows
  for f in $wf_changed; do [ -f "$f" ] && cp "$f" "operations/proposed-workflows/$(basename "$f")" || true; done
  git checkout -- .github/workflows 2>/dev/null || true
  git clean -fdq -- .github/workflows || true
fi
git add -A
```

退避した案は、`workflows` 権限を持つ人間（または PAT）側で反映します。エージェントには手順書で「ワークフローは編集しない」と明記しておくのが先ですが、ガードは最後の砦として必要でした。

## 3. Google のパスワードを変えると、2つの鍵が同じ日に死ぬ

ある日から6日間、Claude の実行が **2秒・1ターン・コスト0** で失敗し続けました。同じ日から日報メール（Gmail SMTP）も
`535-5.7.8 Username and Password not accepted` で失敗。原因はオーナーが **Google アカウントのパスワードを変更した**ことでした。

- Google はパスワード変更時に**発行済みのアプリパスワードをすべて失効**させる → SMTP の鍵が死ぬ
- Claude 側も Google サインインのセッションが切れ、`CLAUDE_CODE_OAUTH_TOKEN` が失効 → 実行が即死

厄介なのは、**失敗を知らせる経路（メール）まで同時に死んだ**ため、6日間誰も気づかなかったことです。
監視が単一障害点でした。対策は「外部の鍵に依存しない通知経路」＝ **GitHub Issue** です（`GITHUB_TOKEN` だけで起票できる）。

```yaml
      - name: Raise alarm issue on failure
        if: failure()
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          EXISTING=$(gh issue list --label 障害 --state open --json number --jq '.[0].number')
          if [ -n "$EXISTING" ]; then gh issue comment "$EXISTING" --body "再発: $GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID"
          else gh issue create --label 障害 --title "🚨 自動実行が失敗しています" --body-file /tmp/body.md; fi
      - name: Close alarm issue on recovery
        if: success()
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: for n in $(gh issue list --label 障害 --state open --json number --jq '.[].number'); do gh issue close "$n"; done
```

Issue はメール通知にもなるので、Gmail の鍵が死んでいても届きます。同じ障害で毎日 Issue を増やさないよう、1件に集約しました。

## 4. 「success なのに何もしていない」は誰も検知しない

鍵を直した後、Actions は毎日 success、メールも毎日届く。ところが記録を追うと、ある晩は成果物を1つコミットしただけで日報なし、
別の晩は**変更ゼロで空のメール**を送っていました。`is_error: false` で終わるので、失敗時アラームも自己修復も動きません。

対策は「成果物の存在を機械で検証して、無ければ失敗にする」ステップです。コミットの後に置くのがポイント（部分成果を捨てない）。

```yaml
      - name: Verify daily report exists
        run: |
          F="operations/daily-reports/$(date +%F).md"
          [ -s "$F" ] || { echo "::error::日報がありません"; exit 1; }
          grep -q "3行サマリー" "$F" || { echo "::error::体裁不備"; exit 1; }
          [ "$(wc -c < "$F")" -ge 800 ] || { echo "::error::中身がありません"; exit 1; }
```

「エージェントが成功と自己申告する」ことと「仕事をした」ことは別です。成功条件は自然言語で頼むだけでなく、検証ステップにします。

## 5. データセンターIP からの HTTP は信用できない（ボット判定）

エージェントに「公開ページが外から見えるか」を `WebFetch` で確認させたところ、**6ページが 404・トップが 403** と報告してきました。
同時刻に家庭回線の未ログインブラウザで開くと**全ページ 200**。販売サイト側が GitHub Actions のIPをボットとして扱っていたのです。

これを「商品が非公開になった」と誤診して方針を変えかけました。教訓は、**到達性の判定は実ブラウザ（家庭回線）に限定し、
クラウドからの 403/404 は「疑いの提起」までに留める**こと。手順書にも明記しました。

## おまけ: `--allowedTools` を省くと書き込みが黙って拒否される

最初の数日は「no changes」で終わっていました。ログの `permission_denials_count: 2` が答えで、`--allowedTools` を明示しないと
Actions 上では Write/Edit/Bash が拒否されます。上の最小構成のように**必要なツールを列挙**してください。

## まとめ（63日で残った運用ルール）

| 落とし穴 | 対策 |
|---|---|
| cron が3.5〜4.5h遅れる | 起点を前倒し／ランナーのTZで営業日付を固定 |
| workflow を含む push が全損 | push 前にワークフロー差分を退避するガード |
| 鍵が同時に死んで沈黙 | 外部の鍵に依存しない通知（GitHub Issue）＋復旧時の自動クローズ |
| success なのに無作業 | 成果物の存在・体裁・分量を検証して失敗にする |
| クラウドからの 403/404 | 実ブラウザで判定。クラウドは疑いの提起まで |

エージェントは「頼めばやる」のではなく、**「検証されることだけをやり続ける」**。63日の学びを1行にするならこれでした。

---
この記事は、AIだけで運営している小さな会社「まめどうぐ製作所」の実運用ログから書いています。
二層運用（クラウド＋ローカル）の競合防止や、エージェントに手順書を守らせる設計は別記事にまとめます。
