# Locust

## Reference

- https://locust.io/
- https://docs.locust.io/en/stable/
- https://github.com/locustio/locust

## About

- Python製の負荷試験ツール
   - Pythonでテストケースを表現することが可能
   - WebベースのUIを備えている
   - CLIだけでも負荷試験を実行可能(headless)
   - 負荷試験中のRPS遷移をグラフで可視化可能(CLI単体実行でもsummaryをHTML出力可能)
   - master/worker 構成で高い負荷(RPS)をかけることが可能

## LocustにおけるRPSを決めるためのパラメータ

### users

- 同時接続ユーザの最大数
    - CLIで `--users=100` の場合、最大同時接続するユーザを100ユーザまで増やすことができる
    - 1000RPSの負荷をかけたい場合、`--users=1000` となる

### spawn rate

- 1秒ごとに増加させる同時接続ユーザ数
    - CLIで `--spawn-rate=1` の場合、同時接続するユーザを秒間1ユーザずつ増やすことになる
    - `--spawn-rate=10` の場合、1000RPSに到達するのは100秒後となる

## master/worker

- https://docs.locust.io/en/stable/running-distributed.html
    - Pythonはプロセスごとに1つ以上のコアを完全に利用することができません([GIL](https://realpython.com/python-gil/) 参照)。
      Locustを実行するマシンのcompute resourceを全て利用するために複数プロセス(`master` プロセスと `worker` プロセス) を起動・同期して負荷をかけることができます。
      また、Locustを実行するマシンを複数台用意しそれぞれでプロセスを起動・同期することも可能です。
    - `master` プロセスと `worker` プロセス は`5557/TCP` で通信します
       - AWSなどのcloud環境のcompute resourceを利用する場合はSecurityGroupなどFWのポートを解放する必要があります
       - `--master-bind-port` でポートを変更することも可能です(worker側も `--master-port` で変更が必要)
    - Usage
        - master
            ```
            # with WebUI
            locust -f locustfile.py --master

            # without WebUI
            locust -f locustfile.py --master --headless

            # wait until worker process start completly with expected process count
            # (require with --headless)
            locust -f locustfile.py --master --headless --expect-workers=5
            ```
        - worker
            ```
            locust -f locustfile.py --worker --master-host=<IPアドレス>
            ```

## CLI単体で実行(headless)

- https://docs.locust.io/en/stable/running-without-web-ui.html
- https://docs.locust.io/en/stable/quickstart.html#direct-command-line-usage-headless

## Locust HTTP Client

- `HttpUser`
    - https://github.com/locustio/locust/blob/320cba0816e1ea38904d70b4672e173a9300747d/locust/user/users.py#L219
    - https://docs.locust.io/en/stable/api.html#httpuser-class
    - `HttpUser` は [`request`](https://requests.readthedocs.io/en/latest/) を使用しているため
      高スループット時にCPU利用効率が悪く想定通りのパフォーマンスがでない場合があります

- `FastHttpUser`
    - https://github.com/locustio/locust/blob/320cba0816e1ea38904d70b4672e173a9300747d/locust/contrib/fasthttp.py#L287
    - https://docs.locust.io/en/stable/increase-performance.html
    - `HttpUser` Classと違い [`geventhttpclient`](https://github.com/geventhttpclient/geventhttpclient) を使用することで
      CPU利用効率があがり `HttpUser` 利用時に比べてより高スループットのパフォーマンスを出せるようになります

## Custom Client

- https://docs.locust.io/en/stable/testing-other-systems.html
    - `User` や `HttpUser` / `FastHttpUser` を継承させることで独自のHTTP Clientを作成することができます

## Custom load shapes

- https://docs.locust.io/en/stable/custom-load-shape.html
    - `locust` 実行時に(WebUIもしくはCLI引数)で指定するUser数やspawn rateでは制御できないユースケースがあります。
      例えば、負荷のスパイクを発生させたり、試験の時間経過によってRPSを増やしたり減らしたり、といったケースです。
    - `LoadTestShape` Classを継承したClassを定義します。LocustはそのClassを自動的に見つけて使用します。
      このClass内で User数とspawn rateをturpleで返す `tick` Methodを定義します(終了時は `None` を返す)。
      Locustは1秒に1回程度 `tick` Methodを呼び出します。

        <details><summary>sample</summary>
        ```
        from locust import FastHttpUser, task, constant_pacing, LoadTestShape
        
        
        class SampleCustomShape(LoadTestShape):
            stages = [
                {"duration": 120, "users": 10, "spawn_rate": 1},
                {"duration": 240, "users": 10, "spawn_rate": 0},
            ]
        
            def tick(self):
                run_time = self.get_run_time()
        
                for stage in self.stages:
                    if run_time < stage["duration"]:
                        tick_data = (stage["users"], stage["spawn_rate"])
                        return tick_data
        
                return None
        
        
        class SampleLoadTest(FastHttpUser):
            wait_time = constant_pacing(1)
        
            host = "http://localhost:3000"
        
            @task
            def task(self):
                name = self.host
                self.client.get("/",)
        ```
        </details>


