# 除錯的技巧

以 [Python binding][appium-python-client] 為例，若收到一個意味不明的錯誤，例如：

    WebDriverException: Message: An unknown server-side error occurred while processing the command.

要如何查出背後的原因？

## 方法

首先，啟動 Appium Server 時要將 log 寫到外部檔案，並加註時間截記 (本地時間)：

    appium ... --log-timestamp --local-timezone --log <LOG_FILE>

事後才能根據時間點，反查當時 server 端是發生什麼錯誤。

例如：

```
  File "test.py", line 21, in <module>
    driver.swipe(0, 0, 1100, 0)
  File "~/.virtualenvs/appium/lib/python2.7/site-packages/appium/webdriver/webdriver.py", line 241, in swipe
    action.perform()
  File "~/.virtualenvs/appium/lib/python2.7/site-packages/appium/webdriver/common/touch_action.py", line 94, in perform
    self._driver.execute(Command.TOUCH_ACTION, params)
  File "~/.virtualenvs/appium/lib/python2.7/site-packages/selenium/webdriver/remote/webdriver.py", line 236, in execute
    self.error_handler.check_response(response)
  File "~/.virtualenvs/appium/lib/python2.7/site-packages/appium/webdriver/errorhandler.py", line 29, in check_response
    raise wde
selenium.common.exceptions.WebDriverException: Message: An unknown server-side error occurred while processing the command.
```

Client 呼叫 `driver.swipe(0, 0, 1100, 0)`，只收到一個 "unknown server-side error"。Server 端的 log 檔如下：

```
...
2016-11-30 17:06:52:668 - [HTTP] --> POST /wd/hub/session/5f29d35c-18d0-4671-9ddd-ddf0d09d349f/touch/perform {"sessionId":"5f29d35c-18d0-4671-9ddd-ddf0d09d349f","actions":[{"action":"press","options":{"y":0,"x":0}},{"action":"wait","options":{"ms":0}},{"action":"moveTo","options":{"y":0,"x":1100}},{"action":"release","options":{}}]}
2016-11-30 17:06:52:669 - [MJSONWP] Calling AppiumDriver.performTouch() with args: [[{"action":"press","option...
2016-11-30 17:06:52:672 - [debug] [AndroidBootstrap] Sending command to android: {"cmd":"action","action":"swipe","params":{"startX":0,"startY":0,"endX":1100,"endY":0,"steps":22}}
2016-11-30 17:06:52:674 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Got data from client: {"cmd":"action","action":"swipe","params":{"startX":0,"startY":0,"endX":1100,"endY":0,"steps":22}}
2016-11-30 17:06:52:675 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Got command of type ACTION
2016-11-30 17:06:52:675 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Got command action: swipe
2016-11-30 17:06:52:690 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Display bounds: [0,0][1080,1776]
2016-11-30 17:06:52:692 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Display bounds: [0,0][1080,1776]
2016-11-30 17:06:52:698 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Returning result: {"status":13,"value":"Coordinate [x=1100.0, y=888.0] is outside of element rect: [0,0][1080,1776]"}
2016-11-30 17:06:52:699 - [debug] [AndroidBootstrap] Received command result from bootstrap
2016-11-30 17:06:52:703 - [HTTP] <-- POST /wd/hub/session/5f29d35c-18d0-4671-9ddd-ddf0d09d349f/touch/perform 500 34 ms - 154
...
```

假設 client 端發生錯誤的時間是 2016-11-30 17:06:52，先從 log 檔案_結尾處_往上找 `2016-11-30 17:06:52` (注意 client 與 server 間是否有時間差)，再往上大概就能找到一些線索...

> <i class="fa fa-lightbulb-o fa-3x"></i>
> 如果 client 跟 server 在不同機器上，建議先校對兩邊的時間，避免時間差增加除錯的難度。

不過 log 那麼長，閱讀上是有一些技巧。觀察上面的例子，不難發現 `[HTTP] -->` 與 `[HTTP] <--` 會成對地出現，這中間的 log 記錄從 server 接到 request、處理、寫出 response 的過程。

    {timestamp} - [HTTP] --> {method} {uri} {params}
    ...
    {timestamp} - [HTTP] <-- {method} {uri} {status} {duration} ms - {bytes}

對照上面的例子：

```
2016-11-30 17:06:52:668 - [HTTP] --> POST /wd/hub/session/5f29d35c-18d0-4671-9ddd-ddf0d09d349f/touch/perform {"sessionId":"5f29d35c-18d0-4671-9ddd-ddf0d09d349f","actions":[{"action":"press","options":{"y":0,"x":0}},{"action":"wait","options":{"ms":0}},{"action":"moveTo","options":{"y":0,"x":1100}},{"action":"release","options":{}}]}
...
2016-11-30 17:06:52:703 - [HTTP] <-- POST /wd/hub/session/5f29d35c-18d0-4671-9ddd-ddf0d09d349f/touch/perform 500 34 ms - 154
```

可以這麼解讀：`driver.swipe(0, 0, 1100, 0)` 觸發了 `/wd/hub/session/.../touch/perform` 這個 POST request，在 server 端內部花了 34 ms 處理，但過程中發生了一些錯誤，所以回應 [HTTP status 500][http-500] (Internal Server Error)，也之所以 client 端會看到 "unknown server-side error"。而問題的原因就在這兩行中間...

> <i class="fa fa-sticky-note-o fa-3x"></i>
> 對於 _server 花多少時間來處理這個 request_ (`{duration} ms`) 的觀察也很重要，有助於釐清時間是花在 client 端還是 server 端。

```
2016-11-30 17:06:52:669 - [MJSONWP] Calling AppiumDriver.performTouch() with args: [[{"action":"press","option...
...
2016-11-30 17:06:52:698 - [AndroidBootstrap] [BOOTSTRAP LOG] [debug] Returning result: {"status":13,"value":"Coordinate [x=1100.0, y=888.0] is outside of element rect: [0,0][1080,1776]"}
2016-11-30 17:06:52:699 - [debug] [AndroidBootstrap] Received command result from bootstrap
```

> <i class="fa fa-lightbulb-o fa-3x"></i>
> 其中 MJSONWP ([Mobile JSON Wire Protocol][mjsonwp]) 是 Appium 內部跟不同平台 driver 間溝通的協定。 

以這個例子而言，是因為 swipe 的座標值超出螢幕範圍：

    Coordinate [x=1100.0, y=888.0] is outside of element rect: [0,0][1080,1776]

## 總結

 * 平時就要將 console log 寫到外部檔，一旦發生問題才有記錄可以反查。
 * 從檔案底部往上找，先找事發的時間點。(注意 client / server 的時間差)
 * 從 `[HTTP] <--` 的結尾，判斷 HTTP status (非 200) 跟 duration 是否有異常，再看 `[HTTP] -->` 與 `[HTTP] <--` 之間的細節。

 [appium-python-client]: https://github.com/appium/python-client
 [http-500]: https://en.wikipedia.org/wiki/List_of_HTTP_status_codes#500
 [mjsonwp]: https://github.com/appium/appium-base-driver/tree/master/lib/mjsonwp

