# 使用 Selenium Wire 在 Python 中进行网页爬虫

[![推广](https://github.com/bright-cn/Awesome-Web-Scraping/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner%20CN.png)](https://www.bright.cn/) 

本指南介绍如何使用 Selenium Wire 进行网页爬虫，并涵盖请求拦截和动态代理轮换等主题。

- [什么是 Selenium Wire？](#什么是-selenium-wire)
- [为什么使用 Selenium Wire？](#为什么使用-selenium-wire)
- [Selenium Wire 的主要功能](#selenium-wire-的主要功能)
    - [访问请求和响应](#访问请求和响应)
    - [拦截请求和响应](#拦截请求和响应)
    - [WebSocket 监控](#websocket-监控)
    - [管理代理](#管理代理)
- [Selenium Wire 中的代理轮换](#selenium-wire-中的代理轮换)
    - [要求](#要求)
    - [第 1 步：随机选择代理](#第-1-步随机选择代理)
    - [第 2 步：设置代理](#第-2-步设置代理)
    - [第 3 步：访问目标页面](#第-3-步访问目标页面)
    - [第 4 步：整合所有代码](#第-4-步整合所有代码)
- [使用 Bright Data 代理轮换代理](#更好的代理轮换方法bright-data-代理)
- [用于网页爬虫时 Selenium 与 Selenium Wire 的对比](#用于网页爬虫时-selenium-与-selenium-wire-的对比)
- [总结](#总结)

## 什么是 Selenium Wire？

[Selenium Wire](https://github.com/wkeeling/selenium-wire) 是 Selenium Python 绑定的扩展，可用于控制浏览器请求。使用 Selenium 时，它允许你直接通过 Python 代码实时拦截和修改请求与响应。

> **注意**：\
> 该库已停止维护，但一些抓取技术和脚本仍在使用它。

## 为什么使用 Selenium Wire？

浏览器存在一些限制，可能会增加网页爬虫的难度。例如，浏览器无法让你设置需要身份验证的代理 URL，也无法即时[轮换代理](https://www.bright.cn/solutions/rotating-proxies)。Selenium Wire 通过像普通用户一样与网站交互，帮助你克服这些限制。

以下是你应使用 Selenium Wire 进行网页爬虫的一些原因：

- **直接访问网络流量**：分析、监控和修改 AJAX 请求与响应，从而高效提取有价值的数据。
- **规避反机器人检测**：[`ChromeDriver`](https://developer.chrome.com/docs/chromedriver/downloads?hl=en) 会暴露反机器人系统用于检测的可识别信息。`undetected-chromedriver` 等技术利用 Selenium Wire 隐藏这些信息并绕过检测机制。
- **增强浏览器灵活性**：传统浏览器依赖固定的启动配置，如需修改则必须重启。Selenium Wire 支持在活动会话中实时更新请求标头和代理设置，因此是动态网页爬虫的理想解决方案。

## Selenium Wire 的主要功能

### 访问请求和响应

Selenium Wire 允许你监控和捕获浏览器的 HTTP/HTTPS 流量，并访问以下关键属性：

| **属性** | **说明** |
| --- | --- |
| `driver.requests` | 按时间顺序返回捕获的请求列表 |
| `driver.last_request` | 返回最近捕获的请求 <br>（这比使用 `driver.requests[-1]` 更高效） |
| `driver.wait_for_request(pat, timeout=10)` | 此方法将等待（等待时间由 `timeout` 参数定义），直到发现与 `pat` 参数所定义模式匹配的请求。该模式可以是子字符串或[正则表达式](https://www.bright.cn/blog/web-data/web-scraping-with-regex)。 |
| `driver.har` | 以 JSON 格式提供已发生 HTTP 事务的 [HAR](https://docs.brightdata.com/api-reference/proxy-manager/get_har_logs) 归档。 |
| `driver.iter_requests()` | 返回已捕获请求的迭代器。 |

Selenium Wire `Request` 对象具有以下属性：

| **属性** | **说明** |
| --- | --- |
| `body` | 请求正文以 `bytes` 形式提供。如果请求没有正文，`body` 的值将为空（例如：`b''`）。 |
| `cert` | 以字典格式返回服务器 SSL 证书的信息（对于非 HTTPS 请求，该字典为空）。 |
| `date` | 显示发出请求的日期和时间。 |
| `headers` | 返回类似字典的请求标头对象（请注意，Selenium Wire 中的标头不区分大小写，并且允许重复）。 |
| `host` | 返回请求主机（例如，`https://www.bright.cn/`）。 |
| `method` | 指定 HTTP 方法（`GET`、`POST` 等）。 |
| `params` | 返回请求参数的字典（请注意，如果请求中多次出现同名参数，则其在字典中的值将是一个列表）。 |
| `path` | 返回请求路径。 |
| `querystring` | 返回查询字符串。 |
| `response` | 返回与请求关联的响应对象（请注意，如果请求没有响应，其值将为 `None`）。 |
| `url` | 返回包含 `host`、`path` 和 `querystring` 的完整请求 URL。 |
| `ws_messages` | 如果请求是 WebSocket（在这种情况下，URL 通常类似于 `wss://`），则 `ws_messages` 将包含已发送和接收的所有 WebSocket 消息。 |

而 `Response` 对象会公开以下属性：

| **属性** | **说明** |
| --- | --- |
| `body` | 响应正文以 `bytes` 形式提供。如果响应没有正文，`body` 的值将为空（例如：`b''`）。 |
| `date` | 显示收到响应的日期和时间。 |
| `headers` | 返回类似字典的响应标头对象（请注意，Selenium Wire 中的标头不区分大小写，并且允许重复）。 |
| `reason` | 返回响应的原因短语，例如 `OK`、`Not Found` 等。 |
| `status_code` | 返回响应状态，例如 `200`、`404` 等。 |

让我们编写一个 Python 脚本来测试此功能：

```python
from seleniumwire import webdriver

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

try:
    # Open the target website
    driver.get("https://brightdata.com/")

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

此代码打开目标网站，并使用 `driver.requests` 捕获请求。随后，它通过 for 循环拦截 `url`、`method` 和 `headers` 等请求属性。

预期结果如下：

![部分已记录的请求](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-98-1024x597.png)

### 拦截请求和响应


Selenium Wire 支持使用拦截器拦截和修改请求与响应。拦截器是网络流量通过浏览器时触发的函数。

有两种独立的拦截器：

* `driver.request_interceptor`：拦截请求并接受一个参数。
* `driver.response_interceptor`：拦截响应并接受两个参数，一个用于原始请求，另一个用于响应。

以下示例展示了如何使用请求拦截器：

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
    driver.get("https://brightdata.com/")

    # Print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Headers: {request.headers}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

此代码片段执行以下操作：

* **拦截器函数**：创建一个拦截器函数，在每个传出请求发出时调用。它使用 `request.headers[]` 为所有传出请求添加自定义标头，并阻止浏览器向 `example.com` 域发送请求。
* **捕获请求**：页面加载完成后，打印所有捕获的请求，包括修改后的标头。

> **注意：**\
> 当页面加载广告、分析脚本或第三方小组件等与你的任务无关的额外资源时，阻止这些请求非常有用。此方法可以提高速度并最大限度减少带宽消耗，从而提升抓取效率。

预期结果应类似如下：

![注意 X-Test-Header](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-99-1024x538.png)

### WebSocket 监控

许多现代网站依靠 [`WebSockets`](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) 与服务器保持实时通信。与传统 HTTP 请求不同，`WebSockets` 会在浏览器与服务器之间建立持续连接，无需反复握手即可实现无缝数据交换。  

由于关键数据通常会通过这些通道传输，拦截 `WebSocket` 流量可直接访问服务器的实时响应，无需通过浏览器进行处理或渲染。

以下是 Selenium Wire `WebSocket` 对象的属性：

| **属性** | **说明** |
| --- | --- |
| `content` | 返回消息内容，其类型可以是 `str`，也可以采用 `bytes` 格式。 |
| `date` | 显示消息的日期和时间。 |
| `headers` | 返回类似字典的响应标头对象（请注意，Selenium Wire 中的标头不区分大小写，并且允许重复）。 |
| `from_client` | 这是一个布尔值：消息由客户端发送时返回 `True`，由服务器发送时返回 `False`。 |

### 管理代理

代理服务器充当你的设备与目标网站之间的中介，并隐藏你的 IP 地址。它们有助于绕过基于 IP 的限制、减少因速率限制而遭到封禁的情况，并访问受地理位置限制的内容，从而实现顺畅的网页爬虫。

让我们在 Selenium Wire 中配置代理：

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

此设置不同于在原版 Selenium 中配置代理，后者需要依赖 Chrome 的 `--proxy-server` 标志。这意味着原版 Selenium 中的代理配置是静态的。设置代理后，它会在整个浏览器会话期间保持生效，且无法在不重启浏览器的情况下修改。此限制可能会带来不便，尤其是在需要动态轮换代理时。

相比之下，Selenium Wire 可以在同一个浏览器实例中动态更改代理。这得益于 `proxy` 属性：

```python
# Dynamically change the proxy
driver.proxy = {
    "http": "<NEW_HTTP_PROXY_URL>",
    "https": "<NEW_HTTPS_PROXY_URL>"
}
```

此外，Chrome 的 `--proxy-server` 标志不支持 URL 中包含身份验证凭据的代理：

```
protocol://username:password@host:port
```

而 Selenium Wire 完全支持需要身份验证的代理，因此更适合用于网页爬虫。

## Selenium Wire 中的代理轮换

让我们创建一个用于代理轮换的 Selenium Wire 项目。这将帮助你在每次请求时更改出口 IP。

### 要求

要完成本指南的这一部分，你需要满足以下前提条件：

* Python 3.7 或更高版本
* [受支持的网页浏览器](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/driver_location/)

首先创建一个虚拟环境目录：

```bash
python -m venv venv
```

在 Windows 上运行以下命令来激活它：

```bash
venv\Scripts\activate
```

在 macOS/Linux 上执行：

```bash
source venv/bin/activate
```

现在安装 Selenium Wire（Selenium 将作为其依赖项自动安装）：

```bash
pip install selenium-wire
```

### 第 1 步：随机选择代理

首先，你需要一个有效代理 URL 列表。你可以使用我们的[免费代理](https://brightdata.com/solutions/free-proxies)列表。将它们添加到列表中，并使用 [`random.choice()`](https://docs.python.org/3/library/random.html#random.choice) 随机选择其中一个元素：

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

调用后，此函数会从列表中随机返回一个代理 URL。

要使其正常运行，请不要忘记导入 `random`：

```
import random
```

### 第 2 步：设置代理

调用 `get_random_proxy()` 函数获取代理 URL：

```python
proxy = get_random_proxy()
```

初始化浏览器实例并设置所选代理：

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

上述代码片段需要导入以下内容：

```python
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
```

要在浏览器会话期间动态更改代理，请改用以下代码：

```python
driver.proxy = {
    "http": proxy,
    "https": proxy
}
```

### 第 3 步：访问目标页面

访问目标网站、提取输出并关闭浏览器：

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

要使其正常运行，请从 Selenium 导入 `By`：

```python
from selenium.webdriver.common.by import By
```

在此示例中，目标页面是 HTTPBin 项目的 [`/ip`](https://httpbin.io/ip) 端点，该页面会返回调用方的 IP 地址。如果一切正常，脚本每次运行时都应打印代理列表中的不同 IP。

### 第 4 步：整合所有代码

以下是应放入 `selenium_wire.py` 文件中的完整 Selenium Wire 代理轮换逻辑：

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

要运行该文件，请执行：

```bash
python3 selenium_wire.py
```

每次运行时，输出应为：

```json
{
  "origin": "PROXY_1:XXXX"
}
```

或者：

```json
{
  "origin": "PROXY_2:YYYY"
}
```

以此类推……

多次运行脚本，你每次都会看到不同的 IP 地址。

## 更好的代理轮换方法：Bright Data 代理

在 Selenium Wire 中手动轮换代理涉及大量样板代码，还需要维护有效代理 URL 列表。你可以改用 Bright Data 的动态代理，它们会自动处理 IP 地址变更。以下是使用方法。

如果你已经有账户，请登录 Bright Data。否则，请免费创建账户。随后你将能够访问以下用户控制面板：

![Bright Data 控制面板](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-100-1024x498.png)

点击“查看代理产品”按钮：

![查看代理产品](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-101.png)

你将被重定向到下方的“代理和抓取基础设施”页面：

![配置住宅代理](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-102-1024x483.png)

向下滚动，找到“[住宅代理](https://www.bright.cn/blog/proxy-101/ultimate-guide-to-proxy-types)”卡片，然后点击“开始使用”按钮：

![住宅代理](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-103.png)

你将进入住宅代理配置控制面板。按照引导式向导，根据你的需求设置代理服务。

![配置住宅代理](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-104.png)

转到“访问参数”选项卡，按如下方式获取代理的主机、端口、用户名和密码：

![访问参数](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-105.png)

请注意，“主机”字段已包含端口。

这些就是构建代理 URL 并在 Selenium Wire 中进行设置所需的全部信息。收集所有信息，并使用以下语法构建 URL：

```
<username>:<password>@<host>
```

例如，在本例中应为：

```
brd-customer-hl_4hgu8dwd-zone-residential:ZZZZZZZZZZ@brd.superproxy.io:XXXXX
```

启用“活动代理”，按照最后的说明操作，一切就准备好了！

![活动代理开关](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-106-1024x164.png)

以下是用于集成 Bright Data 的 Selenium Wire 代理代码片段：

```python
# Bright Data proxy URL
proxy = "brd-customer-hl_4hgu8dwd-zone-residential:ZZZZZZZZZZ@brd.superproxy.io:XXXXX"

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

## 用于网页爬虫时 Selenium 与 Selenium Wire 的对比

总而言之，以下是 Selenium 与 Selenium Wire 的对比：

|     | **Selenium** | **Selenium Wire** |
| --- | --- | --- |
| **用途** | 自动化网页浏览器，以执行 UI 测试和网页交互 | 扩展 Selenium，为检查和修改 HTTP/HTTPS 请求与响应提供额外功能 |
| **HTTP/HTTPS 请求处理** | 不提供对 HTTP/HTTPS 请求或响应的直接访问 | 支持检查、修改和捕获 HTTP/HTTPS 请求与响应 |
| **代理支持** | 代理支持有限（需要手动配置） | 高级代理管理，支持动态设置 |
| **性能** | 轻量且快速 | 由于需要捕获和处理网络流量，速度略慢 |
| **使用场景** | 主要用于 Web 应用程序的功能测试，也适用于基础网页爬虫场景 | 适用于 API 测试、网络流量调试和网页爬虫 |

## 总结

虽然 Selenium Wire 可以高效用于网页爬虫，但它已停止维护，也不是适用于所有场景的通用解决方案。

你可以考虑将原版 Selenium 与专用抓取浏览器结合使用，例如 [Bright Data 的抓取浏览器](https://www.bright.cn/products/scraping-browser)。这是一款可扩展的云浏览器，可与 [Playwright](https://www.bright.cn/products/scraping-browser/playwright)、[Puppeteer](https://www.bright.cn/products/scraping-browser/puppeteer)、[Selenium](https://www.bright.cn/products/scraping-browser/selenium) 等工具配合使用。它能为每个请求无缝轮换出口 IP，同时处理浏览器指纹、重试、验证码破解等任务。立即试用，消除封禁问题并优化你的抓取工作流。
