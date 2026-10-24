# web scraping java spring boot: 从 Jsoup demo 到生产级爬虫——什么时候该接抓取 API，以及 ScraperAPI 全套餐价格、credit 成本与选型建议

用 Spring Boot 搭一个能跑的网页爬虫，大多数 Java 开发者一两个小时就能搞定。真正耗时间的是后面的事：目标站上了 Cloudflare，数据中心 IP 被整段拉黑，页面数据藏在 React 渲染后的 DOM 里。这时候你有两条路——自建代理池加无头浏览器集群，或者把取页这一层外包给专门的抓取 API。这篇文章按实际决策顺序讲清楚三件事：Java 技术栈怎么选、什么时候值得引入 ScraperAPI 这类抓取 API、以及它的 8 档套餐在 credit 计费规则下真实花多少钱。

> **先说结论**：大多数个人项目和中小规模抓取任务，Hobby 套餐（$49/月，10 万 credits，20 并发）够用。注册后有每月 1,000 credits 的免费层可以试，前 7 天还有 5,000 credits 的试用额度，不需要信用卡。付费后 7 天内不满意可以无理由退款。👉 [查看 ScraperAPI 全部套餐与当前价格](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Spring Boot 做网页抓取，技术栈怎么选

选型只看一个判断：你要的数据在不在原始 HTML 源码里。右键"查看网页源代码"能搜到目标内容，用 Jsoup；搜不到，再上浏览器自动化。

主流选择就四个，各管一段：

| 库 | 能力边界 | 适合场景 |
| --- | --- | --- |
| Jsoup | HTML 解析 + HTTP 请求一体，不执行 JS | 静态页、列表页、详情页 |
| Playwright for Java | 无头 Chromium/Firefox/WebKit | SPA、JS 渲染、登录流程、无限滚动 |
| Selenium WebDriver | 全功能浏览器自动化 | UI 测试；抓取场景更慢更重 |
| HtmlUnit | 纯 Java 无头浏览器 | 内网、遗留系统，扛不住现代 JS 框架 |

这套选型的核心逻辑，一篇流传较广的 Spring Boot 爬虫教程概括得很直白：先用 Jsoup，如果目标数据不在"查看源代码"的原始 HTML 里，再换 Playwright——Jsoup 能干的活别动用浏览器，两者解析速度差两个数量级（该教程给的量级是静态解析约 2ms/页，浏览器方案约 400ms/页）。

项目骨架没什么悬念：在 start.spring.io 生成 Spring Boot 3.3.x 项目，Java 21（这是 Spring Boot 3.3+ 的硬性要求），依赖 spring-boot-starter-web、jsoup（教程使用的最新版为 1.18.3）、Lombok，需要 JS 渲染再加 Playwright Java（1.44.0）。典型架构是 Controller → Service → Repository 三层，`@Scheduled` 做定时调度，`@RestController` 直接把抓取结果暴露成 API——这些能力 Python 脚本要么没有，要么得另外拼装。ScrapingBee 工程博客对什么时候该用框架说过一句很到位的话：当抓取需要调度、错误恢复、数据持久化和 API 的时候，你就需要一个真正的应用框架，那就是 Spring Boot。

到这一步为止，一切顺利。问题是生产环境不会停在静态页。

## Spring Boot 爬虫真正卡住人的四个地方

demo 和生产爬虫之间隔着的，是目标网站的反爬体系。具体表现为四类问题：

1. **IP 封锁**。稍大一点的电商或数据站点会把常见数据中心 IP 段整段拉黑，你本机跑得通，部署到云服务器立刻 403。
2. **反爬防护**。Cloudflare、Datadome、PerimeterX 这类方案会弹验证码或直接拦截无头浏览器指纹，Selenium 的默认特征尤其容易被识别。
3. **JS 渲染**。数据只在渲染后的 DOM 里，意味着每次请求都要起一个浏览器实例——内存、速度、稳定性全都是成本。
4. **频控与会话**。单 IP 请求太快被封，分页和多步流程又需要粘住同一个 IP，代理轮换策略得自己写。

每一条单拿出来都可解，加在一起就是一个持续的工程负担。自己维护住宅代理池，月成本很容易超过一套抓取 API 的订阅费，而且代理质量衰减、验证码服务对接这些维护工作不会停。这正是抓取 API 存在的位置：你发一个带目标 URL 的请求，它负责代理轮换、验证码、JS 渲染，把 HTML 还给你，Spring Boot 里的解析和业务逻辑照旧用 Jsoup 写。

## 怎么把 ScraperAPI 接进 Spring Boot 项目

接入方式有两种，官方 Java SDK 更顺手。Maven 依赖一行：

xml
<dependency>
  <groupId>com.scraperapi</groupId>
  <artifactId>sdk</artifactId>
  <version>1.2</version>
</dependency>


SDK 是 fluent builder 风格，API key 按官方建议从环境变量 `SCRAPERAPI_API_KEY` 读取。最基本的用法：

java
ScraperApiClient client = new ScraperApiClient(System.getenv("SCRAPERAPI_API_KEY"));
String html = client.get("https://example.com/products")
    .countryCode("us")
    .result();


参数全部靠链式调用追加：JS 渲染的页面加 `.render(true)`；分页或多步流程用 `.sessionNumber(n)` 粘住同一代理 IP（会话在最后一次使用后 15 分钟过期）；模拟移动端用 `.deviceType("mobile")`；抓 Amazon、Google 这类受支持平台时 `.autoparse(true)` 直接返回结构化 JSON，Jsoup 解析那步都可以省掉。批量超过 20 个 URL 时别用阻塞的 `.result()` 占满线程池，把任务提交到官方异步端点 `https://async.scraperapi.com/jobs`，拿到任务 ID 后轮询结果。

不想引 SDK 也行——ScraperAPI 就一个端点，GET `http://api.scraperapi.com?api_key=你的KEY&url=目标地址` 即可，加参数就是追加 `&render=true&country_code=us` 这样的查询串，任何 HTTP 客户端都能调。

官方文档里明确写了三个坑，提前知道能省不少调试时间：

- **默认别开 render。** 官方原话是先试不加 render 的请求，因为它会增加成本和延迟。每个请求 +10 credits，这笔账后面细算。
- **客户端超时别设太低。** ScraperAPI 内部对失败请求的重试最长持续 70 秒，SDK 默认自己重试 3 次；把 HTTP 客户端超时设成 5 秒，会把本来能成功的请求判成假失败。
- **`.premium(true)` 和 `.ultraPremium(true)` 互斥**，官方明确说不要同时链式调用，且 ultraPremium 与自定义 headers 不兼容。

升级阶梯官方也给得很清楚：标准请求 1 credit，被封就上 render（+10），还不行上 premium（+10），最难缠的站才动 ultraPremium（+30）。👉 [注册免费试用，用 5,000 credits 先测你目标站点的实际消耗](https://www.scraperapi.com/signup?fp_ref=coupons)

## ScraperAPI 全套餐对比

官网当前展示 8 档方案，年付一律比月付省 10%。所有付费套餐都包含 JS 渲染、住宅与移动 IP、验证码与反爬检测、JSON 自动解析、自定义会话、自动重试、不限带宽和 99.9% 在线率保证——差别只在量：

| 套餐 | 月付 | 年付（折算每月） | Credits/月 | 并发 | 购买入口 |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | $0 | 1,000 | 5 | [注册免费账户](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [开通 Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [开通 Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [开通 Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [开通 Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [开通 Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [开通 Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 定制报价 | 定制报价 | 22,000,000+ | 500+ | [联系获取报价](https://www.scraperapi.com/pricing?fp_ref=coupons) |

低档套餐有两个容易忽略的限制：Hobby 和 Startup 的地理定位只覆盖美国和欧洲（Business 起才支持国家级定位），分析面板也只保留最近 30 天的数据。需要指定其他国家代理的话，直接上 Business。

## 真实成本：credit 怎么烧，额度用完怎么办

ScraperAPI 按 credit 计费，而 credit 消耗因目标站而异——这是多数测评文章没讲透、但对预算影响最大的部分。

官方定价口径是：普通页面 1 credit，**Amazon 每次抓取消耗 5，Google 和 Bing（含子域）25，LinkedIn 30**；Cloudflare、Datadome、PerimeterX 防护下的站点，每绕过一次加 10 credits。也就是说，同样 10 万 credits，抓普通网页是 10 万次，抓 Google 搜索结果只有 4,000 次，差 25 倍。动手买套餐之前，先在 dashboard 里用 Domain Cost Estimator 查一下你目标域名的实际单价，还可以给单次请求设 `max_cost` 上限，防止某个意外请求烧掉一堆 credits。

额度用完之后的行为也分档：

- 默认是**暂停抓取**，等下个周期额度刷新，不会偷偷扣钱。
- Hobby、Startup、Business、Scaling、Professional 五档可以开启 **auto-upgrade**：credits 用尽自动升到下一档（需要主动打开这个开关）。官方提示升级后单 credit 单价通常更优，所以手动升级到更高档本身也是省钱的路径之一。
- Scaling、Professional、Advanced、Enterprise 四档支持 **pay-as-you-go**：额度用完后按固定单价继续抓，上限是你自设的月度消费金额，不会超出。

还有一个好消息：超出并发限制不罚钱。超额请求会收到 429 响应，直接被拒收而不是排队，官方明确表示不因此额外收费。写 Spring Boot 客户端时把 429 当成"稍后重试"信号处理就行，配合 SDK 的 `.retry(n)` 参数可以自动化。

年付和月付的实际差额也值得算一下。以 Hobby 为例，月付一年 $588，年付 $529.20，省 $58.80；Advanced 档年付每月省 $197.50，一年就是 $2,370。规律很简单：用量确定就年付，还在评估期就月付。拿不准的话，付费后 7 天内可以无理由退款，随时取消订阅也不收费（取消会在下个续费日生效，周期内剩余 credits 可以继续用完）。👉 [对比月付与年付价格，选适合你的计费周期](https://www.scraperapi.com/pricing?fp_ref=coupons)

## 按场景选套餐：决策规则

不需要爬得很深就能做出选择，按月 credits 消耗倒推即可：

- **月消耗在 10 万 credits 以内**（大约每天 3,000 个普通页面，或 600 个 Amazon 页面）：Hobby，$49 起步，别多花。
- **月消耗 10 万–100 万**，或者需要 20 个以上并发线程加速批量任务：Startup，100 万 credits 加 50 并发。
- **需要美国、欧洲以外国家的代理定位**：直接从 Business（$299）起步，这是第一个支持国家级定位的档位。
- **大批量常态化抓取**（数百万 credits/月）：Scaling 及以上，从这档开始有 pay-as-you-go，突发用量不会让抓取中断。

也有明确不该买的情况。如果目标站点全是静态页、没有反爬、每天几百个请求，Jsoup 加一台便宜 VPS 就解决了，抓取 API 属于花冤枉钱。反过来说，如果你的爬虫天天在和 Cloudflare 搏斗，维护代理池的时间成本早就超过每月几十美元的订阅费——这算的是总账，单独比单价没有意义。

## 常见问题

**ScraperAPI 有免费版吗？** 有。注册即得每月 1,000 免费 credits，最多 5 个并发连接，面向小型抓取项目；注册后前 7 天另有 5,000 次免费请求可用于更大规模测试，官方文档还提到测试额度不够可以联系支持申请补充。

**不想引 SDK，Spring Boot 里最省事的调用方式是什么？** 直接用 `RestClient` 或 `RestTemplate` GET `http://api.scraperapi.com`，带上 `api_key` 和 `url` 两个查询参数，返回的就是目标页 HTML。功能参数（render、country_code 等）全部通过 URL 查询串追加。

**批量抓取会不会把 Java 线程池占满？** 会，如果用阻塞的 `.result()`。超过 20 个 URL 的批量任务改走异步端点：POST JSON 到 `https://async.scraperapi.com/jobs`，响应里带任务 ID 和 statusUrl，轮询取结果即可。Java 21 的虚拟线程也能缓解，但异步端点更省心。

**试用要绑信用卡吗？** 7 天试用 5,000 credits，不需要信用卡。这基本消除了"忘记取消被扣费"的风险。

## 下单前的最后一步

别跳过试跑这一步。同样的套餐，抓静态博客站和抓防护理站点的消耗能差十几倍，你的目标域名实际烧多少 credits，决定了 Hobby 和 Startup 谁才够用。注册免费层，跑通你的 Spring Boot 爬虫，记下 Domain Cost Estimator 给出的域名单价，再按上面的决策规则选档——整个验证过程用免费额度就能完成，不用先花一分钱。👉 [用免费额度试跑 ScraperAPI，确认成本后再选套餐](https://www.scraperapi.com/?fp_ref=coupons)
