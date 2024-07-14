---
title: "GitHub ActionsのCI/CDをSlack通知する際に考えたこと"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "slackapp", "cicd"]
published: true
published_at: 2024-07-14 23:50
---

CI/CDやテスト実行など、GitHub Actionsで自動化したワークフロー結果をSlack通知する方法は色々ある。

大まかに言えば次の2択だが、2つ目の方法で必要なものを必要な時に通知する方が良い。
1. GitHubにインストールしたSlackインテグレーションを使う
    - [Setting - Applications](https://github.com/settings/installations/)で設定する
2. Slack Appを作成してGitHub Actionsから実行する
    - [Your Apps](https://api.slack.com/apps)から作成する

今回はSlackが提供している[slackapi/slack-github-action](https://github.com/slackapi/slack-github-action)を使うために、`workflow_call`方式でまとめる方針をとった。
再利用性を検討したかったことが1番の理由でした。

## Slack Appの作成
詳細は省略するが、上記リンクから`Create New App`を進めて`Bot User OAuth Token`を発行できれば完了。OAuth Scopeは`chat:write`でOK。

## GitHub Actions変数の設定
- `SLACK_NOTIFY_CHANNEL_ID`: 通知先のSlackチャネルID
- `SLACK_TOKEN_GITHUB_NOTIFY`: Slack Appの`Bot User OAuth Token`
    - Webhook URLはセキュリティ上、センシティブな実装になるので避ける

今回作成した`workflow_call`ではこの2つの変数を使っている。
以下、payloadブロックはSlack仕様に則るので省略するが`${{ github.repository }}`等の変数をそのまま記述できるので、通知内容にURLリンクを含めることもできるので、通知メッセージのアクセシビリティに良かった。

```yaml
name: Slack Notification Workflow

on:
  workflow_call:
    inputs:
      exitCode:
        required: true
        type: string
      slackChannelId:
        required: true
        type: string

jobs:
  notify:
    runs-on: ubuntu-latest

    steps:
      - name: Notify Slack - Success
        if: ${{ inputs.exitCode == 'success' }}
        uses: slackapi/slack-github-action@v1.26.0
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_TOKEN_GITHUB_NOTIFY }}
        with:          
          channel-id: ${{ vars.SLACK_NOTIFY_CHANNEL_ID }}
          payload: |
            # Success Notification

      - name: Notify Slack - Failure
        if: ${{ inputs.exitCode != 'success' }}
        uses: slackapi/slack-github-action@v1.26.0
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_TOKEN_GITHUB_NOTIFY }}
        with:
          channel-id: ${{ vars.SLACK_NOTIFY_CHANNEL_ID }}
          payload: |
            # Failure Notification
```

```
# payload部分のURLリンク例: workflow_call実行時のコミットURL
# <url|display_name>という形式はSlack仕様
<https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}/|actions #${{ github.run_id }}>
```

workflow_callが実行された時点で通知したいGitHub Actions結果はわかっている想定なので、`exitCode`引数を必須としている。また、前提となるGitHub Actions変数2つもworkflow_call側で参照するので、workflow_callの呼び出し元が意識することはない。

## workflow_callの呼び出し
ややGitHub Actionsのtipsが必要だったが、どれも基本的なことの組み合わせで落ち着いた。
以下、ワークフロー結果を通知したいworkflow_call呼び出し元でエラーが発生する場合の例。

```yaml
name: test dispatch event

on:
  workflow_dispatch:

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      exitCode: ${{ steps.saveExitCode.outputs.exitCode }}

    steps:
      # - name: Checkout
      #   id: checkout
      #   uses: actions/checkout@v4
      - name: Dummy Checkout
        id: checkout
        continue-on-error: true
        run: |
          exit 1

    # 本来は処理のステップがここに追加されるが、ここではエラー発生時の再現としてexit 1を実行している

      - name: Save exitCode
        id: saveExitCode
        run: |
          echo "exitCode=${{ steps.checkout.outcome }}" >> $GITHUB_OUTPUT

  call_SlackNotify:
    needs: setup
    if: always()
    uses: ${org}/${repo}/.github/workflows/slack_notify.yml@${branch_name}
    secrets: inherit
    with:
      exitCode: ${{ needs.setup.outputs.exitCode }}
      slackChannelId: ${{ vars.SLACK_NOTIFY_CHANNEL_ID }}
```

- 処理ステップのエラー有無に関わらず`call_SlackNotify`ジョブは実行したいため、通知したいステップに`continue-on-error: true`を設定する
    - call_SlackNotifyジョブに`if: always()`は不要だが分かりやすさ重視
- outputsに処理ステップの結果(`outcome`)を保存し、`call_SlackNotify`ジョブの引数とする
    - `continue-on-error: true`の場合、`steps.<step_id>.conclusion`の値は必ず`success`になるので`outcome`を参照する必要あり
- 呼び出し元リポジトリに設定したGitHub Actions変数をworkflow_callで使うため、`secrets: inherit`が必要

## Slack通知の結果

![](/images/slack_notify.png)

## 終わりに
Slack Appsを作成するためにAPIサーバが必要かと誤解していたので、`slackapi/slack-github-action`を効率よく使う方法を考える機会になって良かった。通知内容(payload)の形式を改善して利便性を高める余地はありそうだが、奥が深すぎて私はすぐ引き返しました。。

↓Block Kit Builderから詳しく見られます。
参考：[Reference: blocks](https://api.slack.com/reference/block-kit/blocks)
