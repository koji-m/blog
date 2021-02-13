---
title: "SlackでBigQueryの構文エラー内容を共有する"
date: 2021-02-13T15:19:03+09:00
draft: true
---

会社ではあるグループにメンションして質問すると犬が有識者を連れてきたり、朝会の時間になるとムックが知らせてくれたりする。
また、システムの障害を通知してくれるなどサービスの安定稼働において重要な役割を担っていたりもする。
Slackを仕事で使い始めて約2年くらいになるが、これらの機能を提供するSlack Appsの仕組みについて何も知らずに使い続けてきたが、流石にエンジニアとしてSlack Appsくらい必要なときにサクッと作れないとまずいかなと思い１つアプリを作ってみた。

## SlackでBigQueryの構文エラーを共有する

最近、非エンジニアの方も普通にBigQueryでSQL書いてデータを見たりする場面がある。SQLの構文エラーの原因がわからない場合、WebUIのスクショをSlackに添付したりして質問するかたちになるが、問題のSQLだけSlackに貼ってエラー内容については自動で共有できたら楽だろうなと思った。

そこで、SlackにSQLを投稿するとそのクエリに対するBigQueryのDry Run結果を表示するSlack Appsを作った。

使い方としては、/bq_checkというslashコマンドを登録しておいて以下の様に入力する。

![slack input command](/images/slack-input-command.png)

するとこんな感じで、構文エラーがある場合はエラーの内容が表示され、エラーが無い場合は処理されるバイト数が表示される。

![slack bq check](/images/slack-bq-check.png)

## アプリの構成

Slack Appsの作り方を調べてみると、リクエストとレスポンスについて同期型と非同期型の２パターンがある。

### 同期型

スラッシュコマンドを打つとアプリのAPIを提供するサーバにHTTPリクエストを投げて、そのレスポンスのボディのデータをSlackのタイムラインに返すもの。

![Slack apps sync](/images/slack-apps-sync.png)

Slackの制約によりレスポンスは3秒以内に返さないとエラーとなってしまうので、今回の用途では要件を満たさない。

### 非同期型

スラッシュコマンドを打った際、HTTPリクエストのペイロード(JSON形式)に`response_url`というチャンネル投稿用のエンドポイントのURLが渡される。
そこで、スラッシュコマンドへのレスポンスはすぐに返しておいて、サーバのバックエンドでの処理結果は処理完了次第この`response_url`にPOSTすれば3秒以上掛かる処理の結果もSlackへ返すことができる。

![Slack apps async](/images/slack-apps-async.png)

BigQueryへのDry Run実行には3秒以上掛かることがあるので今回はこちらのパターンを採用。

## 構成

BigQueryへのAPIコールがメインなのでAPIサーバはCloud Functionsを使った。また、非同期型の構成を取るため、APIリクエストのエンドポイント用インスタンスからCloud Pub/Sub経由でバックエンド処理・レスポンス用のインスタンスをキックする構成とした。([参考: Slack のチュートリアル - Slash コマンド](https://cloud.google.com/functions/docs/tutorials/slack?hl=ja))

![Slack BigQuery Checker](/images/slack-bq-diagram.png)

#### APIエンドポイントのコード

リクエストのペイロードに含まれる投稿メッセージと`response_url`をCloud Pub/Subのトピックにpublishしてすぐにレスポンスを返すだけ。

```python
from google.cloud import pubsub_v1
from slack.signature import SignatureVerifier

publisher = pubsub_v1.PublisherClient()

def verify_signature(request):
    request.get_data()

    verifier = SignatureVerifier(os.environ['SLACK_SECRET'])

    if not verifier.is_valid_request(request.data, request.headers):
        raise ValueError('Invalid request/credentials.')


def bq_check(request):
    if request.method != 'POST':
        return 'Only POST requests are accepted.', 405

    verify_signature(request)

    query = request.form['text']
    response_url = request.form['response_url']

    topic_path = publisher.topic_path(os.environ['PROJECT_ID'], os.environ['TOPIC'])

    message_json = json.dumps({
        'query': query,
        'response_url': response_url
    })
    message_bytes = message_json.encode('utf-8')

    try:
        publish_future = publisher.publish(topic_path, data=message_bytes)
        publish_future.result()
        return ('Query checking...', 200)
    except Exception as e:
        print(e)
        return (e, 500)
```

#### バックエンド処理のコード

Pub/Subから渡された投稿メッセージからSQLを抽出してBigQueryのDry Runを実行。結果を`response_url`として渡されたURLにPOSTする。
Slackの投稿メッセージの中で\`\`\`と\`\`\`の間にSQLを入力することでチェック対象のクエリとして認識されるようにした。

```python
from google.cloud import bigquery
import requests

client = bigquery.Client()

def execute_dry_run(message):
    job_config = bigquery.job.QueryJobConfig(dry_run=True, use_legacy_sql=False)
    m = re.match(r'.*\`\`\`(.*)\`\`\`',
                 message.replace('\n', ' '))
    query = m[1]
    try:
        query_job = client.query(query, job_config=job_config)
    except Exception as ex:
        message = {
            'response_type': 'in_channel',
            'text': message,
            'attachments': [
                {
                    'color': '#dc143c',
                    'text': error['message']
                } for error in ex.errors
            ]
        }
    else:
        size = query_job.total_bytes_processed
        if size < 1024:
            processed_size = f"{size}B"
        elif size < 1024 ** 2:
            processed_size = f"{size / 1024}KB"
        elif size < 1024 ** 3:
            processed_size = f"{size / (1024 ** 2)}MB"
        elif size < 1024 ** 4:
            processed_size = f"{size / (1024 ** 3)}GB"
        else:
            processed_size = f"{size / (1024 ** 4)}TB"

        message = {
            'response_type': 'in_channel',
            'text': message,
            'attachments': [
                {
                    'color': '#3367d6',
                    'text': f"{processed_size} 処理されます"
                }
            ]
        }

    return json.dumps(message)

def dry_run(event, context):
    data = json.loads(base64.b64decode(event['data']).decode('utf-8'))
    res = execute_dry_run(data['query'])
    requests.post(data['response_url'], res)
```

## 最後に

特にハマりどころも無くすんなり実装できた。Cloud FunctionsとPub/Subでの実装がシンプルかつ、Cloud Loggingを活用することでデバッグも簡単に行えるのでスムーズに動作確認できた。

今回作ったアプリを使えばSlackでお手軽にBQクエリのレビューができるようになるので、会社全体のBQ力を爆上げしていきましょう。(まずは自分から...)
