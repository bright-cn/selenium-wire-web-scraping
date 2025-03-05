# 使用 Selenium Wire 进行 Python 网络爬虫

[![推广图](https://github.com/bright-cn/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://www.bright.cn/)

本指南介绍了如何在 Python 中使用 Selenium Wire 进行网络爬虫，并涵盖诸如拦截请求和动态代理轮换等主题。

- [什么是 Selenium Wire？](#什么是-selenium-wire)
- [为什么使用 Selenium Wire？](#为什么使用-selenium-wire)
- [Selenium Wire 的主要功能](#selenium-wire-的主要功能)
    - [访问请求与响应](#访问请求与响应)
    - [拦截请求与响应](#拦截请求与响应)
    - [WebSocket 监控](#websocket-监控)
    - [管理代理](#管理代理)
- [在 Selenium Wire 中进行代理轮换](#在-selenium-wire-中进行代理轮换)
    - [环境要求](#环境要求)
    - [步骤 1：随机选择代理](#步骤-1随机选择代理)
    - [步骤 2：设置代理](#步骤-2设置代理)
    - [步骤 3：访问目标页面](#步骤-3访问目标页面)
    - [步骤 4：整合所有步骤](#步骤-4整合所有步骤)
- [使用 Bright Data 代理进行代理轮换](#使用-bright-data-代理进行代理轮换)
- [Selenium 与 Selenium Wire 在爬虫中的对比](#selenium-与-selenium-wire-在爬虫中的对比)
- [结论](#结论)

## 什么是 Selenium Wire？

[Selenium Wire](https://github.com/wkeeling/selenium-wire) 是 Selenium Python 绑定的一个扩展，能够对浏览器请求进行控制。它允许直接在使用 Selenium 的同时，从 Python 代码中实时拦截并修改请求与响应。

> **注意**：  
> 该库目前已停止维护，但依然有不少爬虫技术和脚本在使用。

## 为什么使用 Selenium Wire？

传统浏览器具有某些限制，这使得网络爬虫变得较为困难。例如，它们无法在运行时设置带有验证信息的代理 URL，也无法在不重启浏览器的情况下进行 [代理轮换](https://www.bright.cn/solutions/rotating-proxies)。Selenium Wire 通过模拟真实用户的操作来帮助你克服这些限制。

以下是使用 Selenium Wire 进行网络爬虫的一些理由：

- **直接访问网络流量**：可以分析、监控并修改 AJAX 请求和响应，从而高效提取有价值的数据。  
- **规避反爬虫检测**：  
  [`ChromeDriver`](https://developer.chrome.com/docs/chromedriver/downloads?hl=en) 会暴露一些可用于反爬虫系统检测的可识别信息。诸如 `undetected-chromedriver` 等技术依托 Selenium Wire 来对这些信息进行伪装，从而绕过检测机制。  
- **增强浏览器灵活性**：传统浏览器依赖固定的启动配置，修改时需要重启。Selenium Wire 能够在会话中实时更新请求头和代理设置，完美适用于动态网络爬虫场景。

## Selenium Wire 的主要功能

### 访问请求与响应

Selenium Wire 可以监控并捕获浏览器的 HTTP/HTTPS 网络流量，提供对以下关键属性的访问：

| **属性** | **描述** |
| --- | --- |
| `driver.requests` | 以时间顺序返回捕获的请求列表 |
| `driver.last_request` | 返回最近捕获的请求 <br>（比 `driver.requests[-1]` 更高效） |
| `driver.wait_for_request(pat, timeout=10)` | 等待（时间由 `timeout` 参数定义）直到检测到与 `pat` 匹配（可为子字符串或[正则表达式](https://www.bright.cn/blog/web-data/web-scraping-with-regex)）的请求 |
| `driver.har` | 一个 JSON 格式的 [HAR](https://docs.brightdata.com/api-reference/proxy-manager/get_har_logs)（HTTP Archive）日志，包含所有 HTTP 事务 |
| `driver.iter_requests()` | 返回一个可迭代对象，遍历捕获到的请求 |

Selenium Wire 的 `Request` 对象包含以下属性：

| **属性** | **描述** |
| --- | --- |
| `body` | 请求体以 `bytes` 形式呈现。如果请求无请求体则为 `b''` |
| `cert` | 以字典形式返回服务器 SSL 证书信息（非 HTTPS 请求时为空） |
| `date` | 请求发起的时间（datetime） |
| `headers` | 以类字典（case-insensitive）的方式返回请求头，允许重复键 |
| `host` | 返回请求的主机（例如 `https://brightdata.com/`） |
| `method` | 请求方法（GET、POST 等） |
| `params` | 返回请求参数组成的字典（若同名参数多次出现，则字典中的值为列表） |
| `path` | 返回请求路径 |
| `querystring` | 返回查询字符串 |
| `response` | 返回与该请求对应的响应对象（若无响应则为 `None`） |
| `url` | 返回完整的请求 URL，包含 `host`、`path` 和 `querystring` |
| `ws_messages` | 若请求为 WebSocket（URL 通常是 `wss://`），则 `ws_messages` 会包含所有发送和接收的消息 |

而 `Response` 对象则具有下列属性：

| **属性** | **描述** |
| --- | --- |
| `body` | 响应体以 `bytes` 形式呈现。如果响应无响应体则为 `b''` |
| `date` | 响应接收的时间（datetime） |
| `headers` | 以类字典（case-insensitive）的方式返回响应头，允许重复键 |
| `reason` | 返回响应的文本原因，如 `OK`、`Not Found` 等 |
| `status_code` | 返回响应的状态码，如 `200`、`404` 等 |

下面是一个测试此功能的示例 Python 脚本：

```python
from seleniumwire import webdriver

# 使用 Selenium Wire 初始化 WebDriver
driver = webdriver.Chrome()

try:
    # 打开目标网站
    driver.get("https://brightdata.com/")

    # 获取并打印所有捕获的请求
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Method: {request.method}")
        print(f"Headers: {request.headers}")
        print(f"Response Status Code: {request.response.status_code if request.response else 'No Response'}")
        print("-" * 50)

finally:
    # 关闭浏览器
    driver.quit()
```

该代码打开目标网站并通过 `driver.requests` 捕获请求。随后，在循环中打印了一些请求属性，如 `url`、`method` 和 `headers`。

下面是可能得到的部分结果示例：

![部分请求日志](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-98-1024x597.png)

### 拦截请求与响应

Selenium Wire 可以使用拦截器（interceptors）拦截并修改请求或响应，这些拦截器会在浏览器网络流量通过时被触发。

拦截器有以下两种：

- `driver.request_interceptor`：拦截请求，接受单个参数（request）  
- `driver.response_interceptor`：拦截响应，接受两个参数：一个是源请求（request），一个是响应（response）

下面展示一个在请求阶段使用拦截器的示例：

```python
from seleniumwire import webdriver

# 定义请求拦截器函数
def interceptor(request):
    # 给所有出站请求添加自定义请求头
    request.headers["X-Test-Header"] = "MyCustomHeaderValue"

    # 阻断目标域名为 example.com 的请求
    if "example.com" in request.url:
        print(f"Blocking request to: {request.url}")
        request.abort()  # 终止该请求

# 使用 Selenium Wire 初始化 WebDriver
driver = webdriver.Chrome()

# 将拦截器函数分配给 driver
driver.request_interceptor = interceptor

try:
    # 打开一个会产生多次请求的网站
    driver.get("https://brightdata.com/")

    # 打印所有捕获的请求
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Headers: {request.headers}")
        print("-" * 50)

finally:
    # 关闭浏览器
    driver.quit()
```

以上脚本的主要功能：

- **定义拦截器函数**：创建一个拦截器函数，该函数会在每个出站请求时被调用，为所有请求添加自定义请求头 `X-Test-Header`，并且阻断请求目标是 `example.com` 的请求。  
- **捕获请求**：页面加载完毕后，打印所有捕获的请求及其修改后的请求头信息。

> **注意**：  
> 屏蔽不必要的请求（如广告、分析脚本或第三方小组件）能节省带宽并提高爬虫效率。

下面是示例的运行效果截图：

![可以看到添加了 X-Test-Header](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-99-1024x538.png)

### WebSocket 监控

很多现代网站使用 [`WebSockets`](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) 与服务器进行实时通信。与传统的 HTTP 请求不同，`WebSockets` 建立起浏览器和服务器之间的长连接，从而在不需要重复握手的情况下持续交流数据。

由于大量关键数据可能通过这些通道传输，拦截 `WebSocket` 流量能够直接获取实时响应数据，而无需依赖浏览器端的处理或渲染。

Selenium Wire 的 `WebSocket` 对象具有以下属性：

| **属性** | **描述** |
| --- | --- |
| `content` | 消息内容，可能是 `str` 或 `bytes` |
| `date` | 消息发送或接收的时间 |
| `headers` | 以类字典的形式返回此消息的头部信息（忽略大小写，允许重复） |
| `from_client` | 布尔值，若为 `True` 表示消息来自客户端，若为 `False` 表示来自服务器 |

### 管理代理

代理服务器充当你的设备与目标网站之间的中间人，可以隐藏你的真实 IP，突破基于 IP 的访问限制和封锁，同时访问特定地理区域限定的内容。

以下是在 Selenium Wire 中配置代理的示例：

```python
# 设置 Selenium Wire 的选项
options = {
    "proxy": {
        "http": "<YOUR_HTTP_PROXY_URL>",
        "https": "<YOUR_HTTPS_PROXY_URL>"
    }
}

# 使用 Selenium Wire 初始化 WebDriver
driver = webdriver.Chrome(seleniumwire_options=options)
```

与原生 Selenium 不同的是，你无需使用 Chrome 的 `--proxy-server` 启动标志配置代理，也无需重启浏览器。因为在原生 Selenium 中，代理一旦设置就只能在重启后才能变更。而 Selenium Wire 则可以在同一个浏览器实例中动态更换代理：

```python
# 在浏览器会话中动态切换代理
driver.proxy = {
    "http": "<NEW_HTTP_PROXY_URL>",
    "https": "<NEW_HTTPS_PROXY_URL>"
}
```

此外，Chrome 的 `--proxy-server` 标志不支持在 URL 中包含用户名和密码这类带身份验证的代理，而 Selenium Wire 则完全支持。

## 在 Selenium Wire 中进行代理轮换

接下来，让我们构建一个简单的 Selenium Wire 项目来实现代理轮换，让每次请求都能更换出口 IP。

### 环境要求

在开始之前，确保你具备以下条件：

- Python 3.7 或更高版本  
- [受支持的浏览器](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/driver_location/)

首先，创建并进入一个虚拟环境：

```bash
python -m venv venv
```

Windows 系统下激活命令：

```bash
venv\Scripts\activate
```

macOS / Linux 系统下：

```bash
source venv/bin/activate
```

然后安装 Selenium Wire（安装时会自动包含 Selenium 的依赖）：

```bash
pip install selenium-wire
```

### 步骤 1：随机选择代理

首先，需要准备一个可用的代理 URL 列表。你可以使用我们提供的[免费代理列表](https://www.bright.cn/solutions/free-proxies)。将这些代理添加到一个列表中，再用 [`random.choice()`](https://docs.python.org/3/library/random.html#random.choice) 从中随机选择一个：

```python
def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # ...
    ]
    
    # 随机返回一个代理
    return random.choice(proxies)
```

代码中需先 `import random`：

```python
import random
```

### 步骤 2：设置代理

调用 `get_random_proxy()` 函数获取一个随机代理 URL：

```python
proxy = get_random_proxy()
```

然后使用下列步骤初始化并配置浏览器实例：

```python
# Selenium Wire 配置
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# 浏览器配置
chrome_options = Options()
chrome_options.add_argument("--headless")  # 启用无头模式

# 使用配置初始化浏览器
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)
```

上述代码需要以下导入：

```python
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
```

如果想在同一个会话中动态更换代理，可使用：

```python
driver.proxy = {
    "http": proxy,
    "https": proxy
}
```

### 步骤 3：访问目标页面

访问目标网站，提取输出，最后关闭浏览器：

```python
try:
    # 访问目标页面
    driver.get("https://httpbin.io/ip")

    # 提取页面内容
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # 处理浏览器或代理出错
    print(f"Error with proxy {proxy}: {e}")
finally:
    # 关闭浏览器
    driver.quit()
```

其中需导入 `By`：

```python
from selenium.webdriver.common.by import By
```

在上述示例中，我们访问了 HTTPBin 项目的 [`/ip`](https://httpbin.io/ip) 端点，该页面会返回调用方的 IP。若脚本运行正常，每次都能打印出代理列表中不同的 IP。

### 步骤 4：整合所有步骤

以下是完整的代理轮换示例，可保存在 `selenium_wire.py` 文件中：

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
        # 添加更多代理...
    ]
    
    # 随机抽取一个代理
    return random.choice(proxies)
 
# 获取随机代理
proxy = get_random_proxy()

# Selenium Wire 配置
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# 浏览器初始化
chrome_options = Options()
chrome_options.add_argument("--headless")

driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)

try:
    # 访问目标页面
    driver.get("https://httpbin.io/ip")

    # 打印页面主体内容
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    print(f"Error with proxy {proxy}: {e}")
finally:
    driver.quit()
```

使用以下命令运行脚本：

```bash
python3 selenium_wire.py
```

每次执行脚本，输出可能类似于：

```json
{
  "origin": "PROXY_1:XXXX"
}
```

或

```json
{
  "origin": "PROXY_2:YYYY"
}
```

多次运行该脚本，你就会看到不同的 IP 地址。

## 使用 Bright Data 代理进行代理轮换

如果不想维护一堆代理 URL，也不想写大量样板代码，可以使用 Bright Data 的旋转代理。它们会自动更换出口 IP。下面介绍如何操作。

已有账号的用户可直接登录，没有的话可以免费注册一个。登录后进入以下所示的用户面板：

![Bright Data 用户面板](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-100-1024x498.png)

点击“View proxy products”按钮：

![查看代理产品](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-101.png)

系统会跳转到“Proxies & Scraping Infrastructure”页面：

![代理与爬虫基础设施](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-102-1024x483.png)

向下滚动，找到 “[Residential Proxies](https://www.bright.cn/blog/proxy-101/ultimate-guide-to-proxy-types)” 选项卡，并点击“Get started”：

![Residential Proxies](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-103.png)

接下来会来到住宅代理（Residential Proxies）的配置页面，按照引导进行配置：

![配置住宅代理](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-104.png)

然后转到“Access parameters”选项卡，找到你的代理主机、端口、用户名和密码：

![代理访问参数](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-105.png)

“Host” 字段通常已经包含了端口信息。

你只需要把以上信息组合成代理 URL，并在 Selenium Wire 中进行设置。假如示例的代理格式如下：

```
brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX
```

然后在控制台中启用“Active proxy”即可：

![启用代理](https://github.com/bright-cn/selenium-wire-web-scraping/blob/main/Images/image-106-1024x164.png)

最后，用下面的代码片段，将 Bright Data 的代理集成到 Selenium Wire 中：

```python
# Bright Data 代理 URL
proxy = "brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX"

# 配置 Selenium Wire
options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# 初始化 WebDriver
driver = webdriver.Chrome(seleniumwire_options=options)
```

## Selenium 与 Selenium Wire 在爬虫中的对比

下表总结了 Selenium 与 Selenium Wire 的差异：

|  | **Selenium** | **Selenium Wire** |
| --- | --- | --- |
| **定位** | 为测试和网络交互自动化而设计，没有提供底层网络请求的直接访问 | 对 Selenium 进行了扩展，能拦截并修改 HTTP/HTTPS 请求和响应 |
| **HTTP/HTTPS 请求处理** | 无法直接捕获或修改 HTTP/HTTPS 请求与响应 | 能够拦截、修改、捕获所有网络请求与响应 |
| **代理支持** | 代理设置有限，需要使用启动参数，且无法动态切换 | 提供高级代理管理，可动态更换，非常适合轮换场景 |
| **性能** | 相对轻量、速度快 | 因要捕获和处理网络流量导致性能略有下降 |
| **应用场景** | 主要用于网站功能测试，也可做简单爬虫 | 适用于需要看网络请求详情、调试以及复杂爬虫需求的场合 |

## 结论

虽然 Selenium Wire 在网络爬虫方面十分方便，但由于该项目已不再维护，也并非对所有场景都适用。

相较之下，也可以考虑结合原生 Selenium 和专门的云端爬虫浏览器，比如 Bright Data 的 [抓取浏览器](https://www.bright.cn/products/scraping-browser)。它可结合 [Playwright](https://www.bright.cn/products/scraping-browser/playwright)、[Puppeteer](https://www.bright.cn/products/scraping-browser/puppeteer)、[Selenium](https://www.bright.cn/products/scraping-browser/selenium) 等自动化框架使用，不仅能自动切换出口 IP，还能应对浏览器指纹检测、重试、验证码识别等问题，帮你更顺畅地进行大规模爬虫。祝你爬虫工作顺利！
