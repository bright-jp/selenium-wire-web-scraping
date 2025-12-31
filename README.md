# PythonでSelenium Wireを使ったWebスクレイピング

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/) 

このガイドでは、WebスクレイピングにSelenium Wireを使用する方法を説明し、リクエストのインターセプトや動的なプロキシローテーションなどのトピックを扱います。

- [Selenium Wireとは？](#what-is-selenium-wire)
- [なぜSelenium Wireを使うのか？](#why-use-selenium-wire)
- [Selenium Wireの主な機能](#key-features-of-selenium-wire)
    - [リクエストとレスポンスへのアクセス](#access-requests-and-responses)
    - [リクエストとレスポンスのインターセプト](#intercept-requests-and-responses)
    - [WebSocketの監視](#websocket-monitoring)
    - [プロキシの管理](#manage-proxies)
- [Selenium Wireでのプロキシローテーション](#proxy-rotation-in-selenium-wire)
    - [要件](#requirements)
    - [ステップ1: プロキシをランダム化する](#step-1-randomize-proxies)
    - [ステップ2: プロキシを設定する](#step-2-set-the-proxy)
    - [ステップ3: ターゲットページにアクセスする](#step-3-visit-the-target-page)
    - [ステップ4: まとめて実行する](#step-4-put-it-all-together)
- [Bright Data Proxiesによるローテーティングプロキシ](#a-better-approach-to-proxy-rotation-bright-data-proxies)
- [WebスクレイピングにおけるSeleniumとSelenium Wireの比較](#selenium-vs-selenium-wire-for-web-scraping)
- [結論](#conclusion)

## Selenium Wireとは？

[Selenium Wire](https://github.com/wkeeling/selenium-wire) は、SeleniumのPythonバインディング向けの拡張で、ブラウザのリクエストを制御できるようにします。Seleniumを使用しながら、Pythonコードから直接、リクエストとレスポンスの両方をリアルタイムにインターセプトして変更できます。

> **Note**:\
> このライブラリはすでにメンテナンスされていませんが、いくつかのスクレイピング技術やスクリプトでは現在も使用されています。

## なぜSelenium Wireを使うのか？

ブラウザには、Webスクレイピングを難しくするいくつかの制限があります。たとえば、認証付きプロキシURLを設定したり、オンザフライで[プロキシをローテーション](/solutions/rotating-proxies)したりできません。Selenium Wireは、人間のユーザーと同様にサイトとやり取りすることで、これらの制限を克服するのに役立ちます。

WebスクレイピングでSelenium Wireを使用すべき理由の一部は次のとおりです。

- **ネットワークトラフィックへ直接アクセスできる**: AJAXリクエストとレスポンスを分析・監視・変更し、価値あるデータを効率的に抽出できます。
- **アンチボット検知を回避できる**: [`ChromeDriver`](https://developer.chrome.com/docs/chromedriver/downloads?hl=en) は、アンチボットシステムが検知に利用する識別可能な情報を露出します。`undetected-chromedriver` のような技術はSelenium Wireを活用してこれらの情報を隠し、検知メカニズムを回避します。
- **ブラウザの柔軟性が向上する**: 従来のブラウザは固定の起動設定に依存しており、変更するには再起動が必要です。Selenium Wireはアクティブなセッション内で、リクエストヘッダーやプロキシ設定をリアルタイムに更新できるため、動的なWebスクレイピングに最適なソリューションとなります。

## Selenium Wireの主な機能

### リクエストとレスポンスへのアクセス

Selenium WireはブラウザからのHTTP/HTTPSトラフィックを監視・キャプチャでき、次の主要属性へアクセスできます。

| **Attribute** | **Description** |
| --- | --- |
| `driver.requests` | キャプチャされたリクエストのリストを時系列順に報告します |
| `driver.last_request` | 直近でキャプチャされたリクエストを報告します  <br>（これは `driver.requests[-1]` を使うより効率的です） |
| `driver.wait_for_request(pat, timeout=10)` | `timeout` パラメータで定義された時間、`pat` パラメータで定義されたパターン（部分文字列または[正規表現](/blog/web-data/web-scraping-with-regex)）に一致するリクエストが見つかるまで待機します。 |
| `driver.har` | 発生したHTTPトランザクションのJSON形式の[HAR](https://docs.brightdata.com/api-reference/proxy-manager/get_har_logs)アーカイブです。 |
| `driver.iter_requests()` | キャプチャされたリクエストに対するイテレータを返します。 |

Selenium Wireの `Request` オブジェクトには、次の属性があります。

| **Attribute** | **Description** |
| --- | --- |
| `body` | リクエストの本文は `bytes` として提示されます。リクエスト本文がない場合、`body` は空になります（例: `b''`）。 |
| `cert` | サーバーのSSL証明書に関する情報を辞書形式で報告します（非HTTPSリクエストでは空です）。 |
| `date` | リクエストが行われた日時を示します。 |
| `headers` | リクエストのヘッダーの辞書ライクなオブジェクトを報告します（Selenium Wireのヘッダーは大文字小文字を区別せず、重複が許可される点に注意してください）。 |
| `host` | リクエストホストを報告します（例: `https://brightdata.jp/`）。 |
| `method` | HHTPメソッド（`GET`、`POST` など）を指定します |
| `params` | リクエストのパラメータの辞書を報告します（同名のパラメータが複数回出現する場合、辞書内の値はリストになります）。 |
| `path` | リクエストパスを報告します。 |
| `querystring` | クエリ文字列を報告します。 |
| `response` | リクエストに関連付けられたレスポンスオブジェクトを報告します（レスポンスがない場合は値が `None` になります）。 |
| `url` | `host`、`path`、`querystring` を含む完全なリクエストURLを報告します。 |
| `ws_messages` | リクエストがWebSocketの場合（URLが一般的に `wss://` のような形式）、`ws_messages` に送受信されたWebSocketメッセージが含まれます。 |

一方で、`Response` オブジェクトは次の属性を公開します。

| **Attribute** | **Description** |
| --- | --- |
| `body` | レスポンスの本文は `bytes` として提示されます。レスポンス本文がない場合、`body` は空になります（例: `b''`）。 |
| `date` | レスポンスが受信された日時を示します。 |
| `headers` | レスポンスのヘッダーの辞書ライクなオブジェクトを報告します（Selenium Wireのヘッダーは大文字小文字を区別せず、重複が許可される点に注意してください）。 |
| `reason` | `OK`、`Not Found` などのレスポンスの理由フレーズを報告します。 |
| `status_code` | `200`、`404` などのレスポンスステータスを報告します。 |

この機能をテストするPythonスクリプトを書いてみましょう。

```python
from seleniumwire import webdriver

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

try:
    # Open the target website
    driver.get("https://brightdata.jp/")

    # Access and print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Method: {request.method}")
        print(f"Headers: {request.headers}")
        print(f"Response Status Code: {request.response.status_code if request.response else 'No Response'}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

このコードはターゲットサイトを開き、`driver.requests` を使用してリクエストをキャプチャします。その後、forループで `url`、`method`、`headers` などのリクエスト属性をインターセプトして出力します。

想定される結果は次のとおりです。

![Some of the logged requests](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-98-1024x597.png)

### リクエストとレスポンスのインターセプト


Selenium Wireでは、インターセプター（ブラウザを通過するネットワークトラフィックに応じてトリガーされる関数）を使って、リクエストとレスポンスをインターセプトおよび変更できます。

インターセプターは2種類あります。

* `driver.request_interceptor`: リクエストをインターセプトし、単一の引数を受け取ります。
* `driver.response_interceptor`: レスポンスをインターセプトし、発生元のリクエスト用とレスポンス用の2つの引数を受け取ります。

次は、リクエストインターセプターの使い方を示す例です。

```python
from seleniumwire import webdriver

# Define the request interceptor function
def interceptor(request):
    # Add a custom header to all requests
    request.headers["X-Test-Header"] = "MyCustomHeaderValue"

    # Block requests to a specific domain
    if "example.com" in request.url:
        print(f"Blocking request to: {request.url}")
        request.abort()  # Abort the request

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

# Assign the interceptor function to the driver
driver.request_interceptor = interceptor

try:
    # Open a website that makes multiple requests
    driver.get("https://brightdata.jp/")

    # Print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Headers: {request.headers}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

このスニペットの動作は次のとおりです。

* **Interceptor function**: 送信されるすべてのリクエストに対して呼び出されるインターセプター関数を作成します。`request.headers[]` を使って、送信されるすべてのリクエストにカスタムヘッダーを追加します。また、`example.com` ドメインへのブラウザリクエストをブロックします。
* **Captures requests**: ページが読み込まれた後、変更されたヘッダーを含め、キャプチャされたすべてのリクエストが出力されます。

> **Note:**\
> リクエストのブロックは、広告、解析スクリプト、サードパーティ製ウィジェットなど、タスクに不要な追加リソースをページが読み込む場合に有効です。このアプローチは、速度を向上させ、帯域幅の消費を最小化することでスクレイピング効率を高めます。

想定される結果は次のようになります。

![Note the X-Test-Header](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-99-1024x538.png)

### WebSocketの監視

多くのモダンなWebサイトは、サーバーとのリアルタイム通信を維持するために [`WebSockets`](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) に依存しています。従来のHTTPリクエストとは異なり、`WebSockets` はブラウザとサーバー間に継続的な接続を作り、繰り返しのハンドシェイクなしでシームレスなデータ交換を可能にします。  

重要なデータがこれらのチャネルを通って流れることが多いため、`WebSocket` トラフィックをインターセプトすれば、リアルタイムのサーバーレスポンスへ直接アクセスでき、ブラウザ側での処理やレンダリングの必要がなくなります。

以下はSelenium Wireの `WebSocket` オブジェクトの属性です。

| **Attribute** | **Description** |
| --- | --- |
| `content` | メッセージの内容を報告します。`str` または `bytes` 形式になります。 |
| `date` | メッセージの日時を示します。 |
| `headers` | レスポンスヘッダーの辞書ライクなオブジェクトを報告します（Selenium Wireのヘッダーは大文字小文字を区別せず、重複が許可される点に注意してください）。 |
| `from_client` | クライアントが送信したメッセージの場合は `True`、サーバーが送信した場合は `False` を返すbooleanです。 |

### プロキシの管理

プロキシサーバーは、デバイスとターゲットWebサイトの間で中継として機能し、IPアドレスを隠します。IPベースの制限の回避、レート制限によるブロッキングの軽減、ジオロケーション制限コンテンツへのアクセスを可能にし、シームレスなWebスクレイピングを支援します。

Selenium Wireでプロキシを設定してみましょう。

```python
# Set up Selenium Wire options
options = {
    "proxy": {
        "http": "<YOUR_HTTP_PROXY_URL>",
        "https": "<YOUR_HTTPS_PROXY_URL>"
    }
}

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome(seleniumwire_options=options)
```

この設定は、素のSeleniumでプロキシを設定する場合（Chromeの `--proxy-server` フラグに依存する必要がある）とは異なります。これは、素のSeleniumではプロキシ設定が静的であることを意味します。いったんプロキシを設定すると、ブラウザセッション全体で有効になり、ブラウザを再起動しない限り変更できません。この制約は、特に動的なプロキシローテーションが必要な場合に制限となり得ます。

これに対して、Selenium Wireは同一ブラウザインスタンス内でプロキシを動的に変更できる柔軟性を提供します。これは `proxy` 属性のおかげで可能になります。

```python
# Dynamically change the proxy
driver.proxy = {
    "http": "<NEW_HTTP_PROXY_URL>",
    "https": "<NEW_HTTPS_PROXY_URL>"
}
```

さらに、Chromeの `--proxy-server` フラグは、URLに認証情報を含むプロキシをサポートしていません。

```
protocol://username:password@host:port
```

その代わり、Selenium Wireは認証付きプロキシを完全にサポートしているため、Webスクレイピングにはより良い選択肢となります。

## Selenium Wireでのプロキシローテーション

プロキシローテーションのためにSelenium Wireプロジェクトをセットアップしましょう。これにより、リクエストごとに退出IPを変更できるようになります。

### 要件

このガイドのこの部分に従うには、次の前提条件が必要です。

* Python 3.7以上
* [対応しているWebブラウザ](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/driver_location/)

まず、仮想環境ディレクトリを作成します。

```bash
python -m venv venv
```

有効化するには、Windowsでは次を実行します。

```bash
venv\Scripts\activate
```

macOS/Linuxでは次を実行します。

```bash
source venv/bin/activate
```

次にSelenium Wireをインストールします（依存関係としてSeleniumが自動的にインストールされます）。

```bash
pip install selenium-wire
```

### ステップ1: プロキシをランダム化する

まず、有効なプロキシURLのリストが必要です。[free proxies](https://brightdata.jp/solutions/free-proxies) のリストを使用できます。これらをリストに追加し、[`random.choice()`](https://docs.python.org/3/library/random.html#random.choice) を使ってランダムな要素を選択します。

```python
def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # ...
    ]
    
    # Randomize the list
    return random.choice(proxies)
```

呼び出されると、この関数はリストからランダムなプロキシURLを返します。

動作させるために、`random` のimportを忘れないでください。

```
import random
```

### ステップ2: プロキシを設定する

`get_random_proxy()` 関数を呼び出してプロキシURLを取得します。

```python
proxy = get_random_proxy()
```

ブラウザインスタンスを初期化し、選択したプロキシを設定します。

```python
# Selenium Wire configuration with the proxy
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Browser configuration
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run the browser in headless mode 

# Initialize a browser instance with the given configurations
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)
```

上記スニペットには、次のimportが必要です。

```python
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
```

ブラウザセッション中にプロキシを動的に変更するには、代わりに次のコードを使用します。

```python
driver.proxy = {
    "http": proxy,
    "https": proxy
}
```

### ステップ3: ターゲットページにアクセスする

ターゲットWebサイトにアクセスし、出力を抽出してブラウザを閉じます。

```python
try:
    # Visit the target page
    driver.get("https://httpbin.io/ip")

    # Extract the page output
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # Handle any errors that occur with the browser or the proxy
    print(f"Error with proxy {proxy}: {e}")
finally:
    # Close the browser
    driver.quit()
```

動作させるために、Seleniumから `By` をimportしてください。

```python
from selenium.webdriver.common.by import By
```

この例では、宛先ページはHTTPBinプロジェクトの [`/ip`](https://httpbin.io/ip) エンドポイントです。このページは呼び出し元のIPアドレスを返します。すべてが期待どおりに動作すれば、スクリプトは実行ごとにプロキシリストから異なるIPを出力するはずです。

### ステップ4: まとめて実行する

次は、`selenium_wire.py` ファイルに入れるべきSelenium Wireのプロキシローテーションロジック全体です。

```python
import random
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By

def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # Add more proxies here...
    ]
    
    # Randomly pick a proxy
    return random.choice(proxies)
 
# Pick a random proxy URL 
proxy = get_random_proxy()

# Selenium Wire configuration with the proxy
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Browser configuration
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run the browser in headless mode 

# Initialize a browser instance with the given configurations
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)

try:
    # Visit the target page
    driver.get("https://httpbin.io/ip")

    # Extract the page output
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # Handle any errors that occur with the browser or the proxy
    print(f"Error with proxy {proxy}: {e}")
finally:
    # Close the browser
    driver.quit()
```

ファイルを実行するには、次を起動します。

```bash
python3 selenium_wire.py
```

実行ごとに、出力は次のようになります。

```json
{
  "origin": "PROXY_1:XXXX"
}
```

または次のようになります。

```json
{
  "origin": "PROXY_2:YYYY"
}
```

など…

スクリプトを複数回実行すると、毎回異なるIPアドレスが表示されるはずです。

## より良いプロキシローテーションのアプローチ: Bright Data Proxies

Selenium Wireで手動のプロキシローテーションを行うには、多くのボイラープレートコードが必要で、有効なプロキシURLのリストを維持する必要があります。代わりに、IPアドレスの変更を自動的に処理するBright Dataのローテーティングプロキシを使用できます。使用方法は次のとおりです。

すでにアカウントをお持ちの場合はBright Dataにログインしてください。そうでない場合は、無料でアカウントを作成してください。次のユーザーダッシュボードにアクセスできるようになります。

![The Bright Data dashboard](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-100-1024x498.png)

「View proxy products」ボタンをクリックします。

![View proxy products](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-101.png)

次の「Proxies & Scraping Infrastructure」ページにリダイレクトされます。

![Configuring your residential proxies](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-102-1024x483.png)

下にスクロールして「[Residential Proxies](/blog/proxy-101/ultimate-guide-to-proxy-types)」カードを見つけ、「Get started」ボタンをクリックします。

![Residential proxies](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-103.png)

レジデンシャルプロキシの設定ダッシュボードに移動します。ガイド付きウィザードに従い、ニーズに合わせてプロキシサービスを設定してください。

![Configuring your residential proxies](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-104.png)

「Access parameters」タブに移動し、次のようにプロキシのhost、port、username、passwordを取得します。

![access parameter](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-105.png)

「Host」フィールドにはすでにportが含まれている点に注意してください。

これで、プロキシURLを構築してSelenium Wireに設定するために必要なものはすべて揃いました。すべての情報を収集し、次の構文でURLを構築します。

```
<username>:<password>@<host>
```

たとえば、このケースでは次のようになります。

```
brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX
```

「Active proxy」を切り替え、最後の指示に従えば準備完了です。

![Active proxy toggle](https://github.com/luminati-io/selenium-wire-web-scraping/blob/main/Images/image-106-1024x164.png)

以下は、Bright Dataを統合するためのSelenium Wireプロキシスニペットです。

```python
# Bright Data proxy URL
proxy = "brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX"

# Set up Selenium Wire options
options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome(seleniumwire_options=options)
```

## WebスクレイピングにおけるSeleniumとSelenium Wireの比較

要約すると、SeleniumとSelenium Wireの比較は次のとおりです。

|     | **Selenium** | **Selenium Wire** |
| --- | --- | --- |
| **Purpose** | UIテストおよびWeb操作を実行するためにWebブラウザを自動化します | Seleniumを拡張し、HTTP/HTTPSリクエストとレスポンスの検査および変更のための追加機能を提供します |
| **HTTP/HTTPS request handling** | HTTP/HTTPSリクエストやレスポンスへ直接アクセスする機能は提供しません | HTTP/HTTPSリクエストとレスポンスの検査・変更・キャプチャが可能です |
| **Proxy support** | プロキシサポートが限定的です（手動設定が必要） | 動的設定をサポートする高度なプロキシ管理 |
| **Performance** | 軽量で高速です | ネットワークトラフィックのキャプチャと処理のため、やや低速です |
| **Use cases** | 主にWebアプリケーションの機能テストに使用され、基本的なWebスクレイピングにも便利です | APIのテスト、ネットワークトラフィックのデバッグ、Webスクレイピングに有用です |

## 結論

Selenium WireはWebスクレイピングに効率的に使用できますが、メンテナンスされていないソフトウェアであり、万能なソリューションではありません。

代わりに、素のSeleniumと、[Bright DataのScraping Browser](https://brightdata.jp/products/scraping-browser) のような専用のスクレイピングブラウザの利用を検討してください。これは[Playwright](https://brightdata.jp/products/scraping-browser/playwright)、[Puppeteer](https://brightdata.jp/products/scraping-browser/puppeteer)、[Selenium](https://brightdata.jp/products/scraping-browser/selenium)などと連携するスケーラブルなクラウドブラウザです。各リクエストごとに退出IPをシームレスにローテーションしつつ、ブラウザフィンガープリント、リトライ、CAPTCHA解決なども管理します。ブロッキングの問題を解消し、スクレイピングのワークフローを最適化するためにお試しください。