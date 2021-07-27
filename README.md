# crawlergo

![chromedp](https://img.shields.io/badge/chromedp-v0.5.2-brightgreen.svg) ![Chromium version](https://img.shields.io/badge/chromium-79.0.3945.0-important.svg) 

> A powerful browser crawler for web vulnerability scanners

English Document | [中文文档](./README_zh-cn.md)

crawlergo is a browser crawler that uses `chrome headless` mode for URL collection. It hooks key positions of the whole web page with DOM rendering stage, automatically fills and submits forms, with intelligent JS event triggering, and collects as many entries exposed by the website as possible. The built-in URL de-duplication module filters out a large number of pseudo-static URLs, still maintains a fast parsing and crawling speed for large websites, and finally gets a high-quality collection of request results.

crawlergo currently supports the following features:
* chrome browser environment rendering
* Intelligent form filling, automated submission
* Full DOM event collection with automated triggering
* Smart URL de-duplication to remove most duplicate requests
* Intelligent analysis of web pages and collection of URLs, including javascript file content, page comments, robots.txt files and automatic Fuzz of common paths
* Support Host binding, automatically fix and add Referer
* Support browser request proxy
* Support pushing the results to passive web vulnerability scanners

## Screenshot

![](./imgs/demo.gif)

## Installation

**Please read and confirm [disclaimer](./Disclaimer.md) carefully before installing and using。**

1. crawlergo relies only on the chrome environment to run, go to [download](https://www.chromium.org/getting-involved/download-chromium) for the new version of chromium, or just [click to download Linux version 79](https://storage.googleapis.com/chromium-browser-snapshots/Linux_x64/706915/chrome-linux.zip).
2. Go to [download page](https://github.com/0Kee-Team/crawlergo/releases) for the latest version of crawlergo and extract it to any directory. If you are on linux or macOS, please give crawlergo **executable permissions (+x)**.

> If you are using a linux system and chrome prompts you with missing dependencies, please see TroubleShooting below

## Quick Start
### Go！

Assuming your chromium installation directory is `/tmp/chromium/`, set up 10 tabs open at the same time and crawl the `testphp.vulnweb.com`:

```shell
./crawlergo -c /tmp/chromium/chrome -t 10 http://testphp.vulnweb.com/
```


### Using Proxy

```shell
./crawlergo -c /tmp/chromium/chrome -t 10 --request-proxy socks5://127.0.0.1:7891 http://testphp.vulnweb.com/
```


### Calling crawlergo with python

By default, crawlergo prints the results directly on the screen. We next set the output mode to `json`, and the sample code for calling it using python is as follows:

```python
#!/usr/bin/python3
# coding: utf-8

import simplejson
import subprocess


def main():
    target = "http://testphp.vulnweb.com/"
    cmd = ["./crawlergo", "-c", "/tmp/chromium/chrome", "-o", "json", target]
    rsp = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    output, error = rsp.communicate()
	#  "--[Mission Complete]--"  is the end-of-task separator string
    result = simplejson.loads(output.decode().split("--[Mission Complete]--")[1])
    req_list = result["req_list"]
    print(req_list[0])


if __name__ == '__main__':
    main()
```

### Crawl Results

When the output mode is set to `json`, the returned result, after JSON deserialization, contains four parts:

* `all_req_list`： All requests found during this crawl task, containing any resource type from other domains.
* `req_list`：Returns the **current domain results** of this crawl task, pseudo-statically de-duplicated, without static resource links. It is a subset of `all_req_list `.
* `all_domain_list`：List of all domains found.
* `sub_domain_list`：List of subdomains found.



## Parameter Description

* **`--chromium-path Path, -c Path`**    The path to the chrome executable. (**Required**)
* **`--custom-headers Headers`**   Customize the HTTP header. Please pass in the data after JSON serialization, this is globally defined and will be used for all requests. (**Default: null**)
* **`--post-data PostData, -d PostData`**   POST data. (**Default: null**)
* **`--max-crawled-count Number, -m Number`**    The maximum number of tasks for crawlers to avoid long crawling time due to pseudo-static. (**Default: 200**)
* **`--filter-mode Mode, -f Mode`**   Filtering mode, `simple`: only static resources and duplicate requests are filtered.  `smart`: with the ability to filter pseudo-static. `strict`: stricter pseudo-static filtering rules. (**Default: smart**)
* **`--output-mode value, -o value`**   Result output mode, `console`: print the glorified results directly to the screen. `json`: print the json serialized string of all results.  `none`: don't print the output. (**Default: console**)
* **`--output-json filepath`** Write the result to the specified file after JSON serializing it. (**Default: null**)
* **`--incognito-context, -i`**   Browser start incognito mode. (**Default: true**)
* **`--max-tab-count Number, -t Number`**   The maximum number of tabs the crawler can open at the same time. (**Default: 8**)
* **`--fuzz-path`**  Use the built-in dictionary for path fuzzing. (**Default: false**)
* **`--fuzz-path-dict`**  Customize the Fuzz path by passing in a dictionary file path, e.g. /home/user/fuzz_dir.txt, each line of the file represents a path to be fuzzed. (**Default: null**)
* **`--robots-path`** Resolve the path from the /robots.txt file. (**Default: false**)
* **`--request-proxy proxyAddress`** **socks5** proxy address, all network requests from crawlergo and chrome browser are sent through the proxy. (**Default: null**)
* **`--tab-run-timeout Timeout`**   Maximum runtime for a single tab page. (**Default: 20s**)
* **`--wait-dom-content-loaded-timeout Timeout`**  The maximum timeout to wait for the page to finish loading. (**Default: 5s**)
* **`--event-trigger-interval Interval`** The interval when the event is triggered automatically, generally used in the case of slow target network and DOM update conflicts that lead to URL miss capture. (**Default: 100ms**)
* **`--event-trigger-mode Value`** DOM event auto-triggered mode, with `async` and `sync`, for URL miss-catching caused by DOM update conflicts. (**Default: async**)
* **`--before-exit-delay`** Delay exit to close chrome at the end of a single tab task. Used to wait for partial DOM updates and XHR requests to be captured. (**Default: 1s**)
* **`--ignore-url-keywords, -iuk`** URL keyword that you don't want to visit, generally used to exclude logout links when customizing cookies. Usage: `-iuk logout -iuk exit`. (**default: "logout", "quit", "exit"**)
* **`--form-values, -fv`** Customize the value of the form fill, set by text type. Support definition types: default, mail, code, phone, username, password, qq, id_card, url, date and number. Text types are identified by the four attribute value keywords `id`, `name`, `class`, `type` of the input box label. For example, define the mailbox input box to be automatically filled with A and the password input box to be automatically filled with B, `-fv mail=A -fv password=B`.Where default represents the fill value when the text type is not recognized, as "Cralwergo". (**Default: Cralwergo**)
* **`--form-keyword-values, -fkv`** Customize the value of the form fill, set by keyword fuzzy match. The keyword matches the four attribute values of `id`, `name`, `class`, `type` of the input box label. For example, fuzzy match the pass keyword to fill 123456 and the user keyword to fill admin, `-fkv user=admin -fkv pass=123456`. (**Default: Cralwergo**)
* **`--push-to-proxy`** The listener address of the crawler result to be received, usually the listener address of the passive scanner. (**Default: null**)
* **`--push-pool-max`** The maximum number of concurrency when sending crawler results to the listening address. (**Default: 10**)
* **`--log-level`** Logging levels, debug, info, warn, error and fatal. (**Default: info**)
* **`--no-headless`**  Turn off chrome headless mode to visualize the crawling process. (**Default: false**)



## Examples

crawlergo returns the full request and URL, which can be used in a variety of ways:

* Used in conjunction with other passive web vulnerability scanners

  First, start a passive scanner and set the listening address to: `http://127.0.0.1:1234/`

  Next, assuming crawlergo is on the same machine as the scanner, start crawlergo and set the parameters:

  `--push-to-proxy http://127.0.0.1:1234/`

* Host binding (not available for high version chrome)  [(example)](https://github.com/0Kee-Team/crawlergo/blob/master/examples/host_binding.py)

* Custom Cookies  [(example)](https://github.com/0Kee-Team/crawlergo/blob/master/examples/request_with_cookie.py)

* Regularly clean up zombie processes generated by crawlergo [(example)](https://github.com/0Kee-Team/crawlergo/blob/master/examples/zombie_clean.py) , contributed by @ring04h

## TroubleShooting

* 'Fetch.enable' wasn't found

  Fetch is a feature supported by the new version of chrome, if this error occurs, it means your version is too low, please upgrade the chrome version.
  
* chrome runs with missing dependencies such as xxx.so

  ```shell
  // Ubuntu
  apt-get install -yq --no-install-recommends \
       libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 \
       libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 \
       libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 \
       libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 libnss3
       
  // CentOS 7
  sudo yum install pango.x86_64 libXcomposite.x86_64 libXcursor.x86_64 libXdamage.x86_64 libXext.x86_64 libXi.x86_64 \
       libXtst.x86_64 cups-libs.x86_64 libXScrnSaver.x86_64 libXrandr.x86_64 GConf2.x86_64 alsa-lib.x86_64 atk.x86_64 gtk3.x86_64 \
       ipa-gothic-fonts xorg-x11-fonts-100dpi xorg-x11-fonts-75dpi xorg-x11-utils xorg-x11-fonts-cyrillic xorg-x11-fonts-Type1 xorg-x11-fonts-misc -y
  
  sudo yum update nss -y
  ```


* Run prompt **Navigation timeout** / browser not found / don't know correct **browser executable path**

   Make sure the browser executable path is configured correctly, type: `chrome://version` in the address bar, and find the executable file path:

  ![](./imgs/chrome_path.png)

## Bypass headless detect
crawlergo can bypass headless mode detection by default.

https://intoli.com/blog/not-possible-to-block-chrome-headless/chrome-headless-test.html

![](./imgs/bypass.png)


## Follow me

Weibo：[@9ian1i](https://weibo.com/u/5242748339) 
Twitter: [@9ian1i](https://twitter.com/9ian1i)

Related articles：[A browser crawler practice for web vulnerability scanning](https://www.anquanke.com/post/id/178339)
