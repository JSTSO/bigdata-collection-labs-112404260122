# Lab 1 实验报告

## 学生信息

- **姓名：**蒋石涛
- **学号：**112404260122
- **班级：**数据1241
- **GitHub 用户名：**JSTSO
- **仓库地址：**https://github.com/JSTSO/bigdata-collection-labs-112404260122

---

# 任务 1：判断数据加载方式

## 1.1 Device：判断动态加载并找到 API

- 页面：`/devices`
- 数据加载方式：**动态加载**
- 判断依据：

1. 查看网页源代码，搜索设备编号 `D00000001`、设备名称 `循环泵-000001-001`，**均搜索不到**；
   `/devices` 初始 HTML 仅 2949 字节，`<tbody id="device-rows">` 内只有 `正在加载…`。
2. Network → Fetch/XHR 中找到返回设备数据的请求 `/api/devices?page=1&page_size=50`，
   `Content-Type: application/json`。

### API 信息

- Request URL：`https://115.159.159.82/api/devices?page=1&page_size=50`
- Request Method：`GET`
- Query 参数：`page=1`、`page_size=50`
  （可选过滤参数：`company_id`、`device_type`、`status`、`zone`）

### 【截图 1】Device API 发现证据

```text
images/01_device_api.png
```

```markdown
![Device API 发现证据](images/01_device_api.png)
```

---

## 1.2 Company：判断静态加载

- 页面：`/companies`
- 数据加载方式：**静态加载**
- 判断依据：查看网页源代码（11331 字节），搜索企业名称 `临港精工001有限公司` **可以搜到**，
  并可见完整企业记录 `data-company-id="C000001"`，含所属行业、所在区域、设备数量等字段。

### 【截图 2】Company 静态加载证据

```text
images/02_company_source.png
```

```markdown
![Company 静态加载证据](images/02_company_source.png)
```

---

# 任务 2：Device API 分页采集

## 2.1 API 分页结构

| 含义 | JSON 字段 |
|---|---|
| 当前页 | `page` |
| 每页条数 | `page_size` |
| 设备总数 | `total` |
| 设备列表 | `items` |

---

## 2.2 关键代码

```python
# 从第一页响应中读取分页信息
total = data["total"]
page_size = data["page_size"]

total_pages = (total + page_size - 1) // page_size

devices = []

for page in range(1, total_pages + 1):
    r = requests.get(
        api_url,
        params={
            "page": page,
            "page_size": page_size
        },
        timeout=10
    )
    r.raise_for_status()

    page_data = r.json()
    page_items = page_data["items"]
    devices.extend(page_items)
```

---

## 2.3 采集结果

- 每页条数：**50**
- 设备总数：**5000**
- 程序计算得到的总页数：**100**
- 最终采集设备数：**5000**
- 唯一 `device_id` 数：**5000**

> 为什么总页数应由 API 返回的 `total` 和 `page_size` 计算，而不是直接在代码里写死？

答：`total` 是服务端当前真实数据量，会随业务增减。写死页数会导致数据量增大时**少采**（且不会报错，最难发现），
数据量减少时**多采**（请求大量空页，浪费时间且可能被限流）。
用 `(total + page_size - 1) // page_size` 向上取整，数据量变化后脚本仍能一次跑对，无需人工改代码，也可复用到其他分页接口。

---

# 任务 3：Company HTML 解析

## 3.1 企业记录 Selector

- 企业记录 selector：**`article.company-card`**
- 第 1 页企业记录数：**20**

- 主要使用了 `id / class / data-* / 父子层级` 中的哪一种？
  → **class**（`company-card`），配合标签名 `article`；`data-*`（`data-company-id`）用于提取 `company_id`。
- 是否参考过浏览器 Copy selector 或 AI？
  → 参考 Elements 面板的 DOM 观察；**未采用** Copy selector 生成的 `nth-child(...)` 长路径。
- 最终为什么采用现在这个写法？
  → 一条企业记录的最外层节点即 `<article class="company-card" data-company-id="...">`，
  `company-card` 语义明确且整页只用于企业记录，一个 class 即可唯一定位；加 `article` 限定标签防止误伤同名 class。
  整条 selector 仅两个片段、不含位置序号，企业顺序调整或字段缺失都不会失效。

---

## 3.2 关键代码

```python
cards = soup.select("article.company-card")

card = cards[0]

company_id = card.get("data-company-id")
name = card.select_one("h2 a").get_text(strip=True)
status = card.select_one(".badge").get_text(strip=True)
```

```python
details = {}
for item in card.select("dl > div"):
    key = item.select_one("dt").get_text(strip=True)
    value = item.select_one("dd").get_text(strip=True)
    details[key] = value

industry = details.get("所属行业")
zone = details.get("所在区域")
device_count = int(re.search(r"\d+", details.get("设备数量", "0")).group())
```

---

## 3.3 字段来源说明

1. 字段：**`company_id`**
   - 来自：**HTML 属性**
   - 使用的方法：`card.get("data-company-id")`，读取 `<article>` 上的 `data-company-id` 属性。

2. 字段：**`device_count`**
   - 来自：**文本**
   - 使用的方法：`dl > div` 的 `dt/dd` 转字典后取值 `"6 台"`，
     再 `re.search(r"\d+", ...).group()` 取数字并 `int()` 转换。

---

# 任务 4：Company "下一页"分页

## 4.1 下一页元素

- "下一页" selector：**`nav.pagination a:last-child`**
- 第 1 页 `href` 示例：**`/companies?page=2`**

### 【截图 3】Company "下一页"元素

```text
images/03_company_next.png
```

```markdown
![Company 下一页元素](images/03_company_next.png)
```

---

## 4.2 关键代码

```python
next_link = soup.select_one("nav.pagination a:last-child")

next_href = next_link.get("href")

next_url = urljoin(
    current_url,
    next_href
)
```

> 你的程序如何完成"当前页 → 找下一页 href → 请求下一页 → 再解析"的循环？

答：`for page_no in range(1, 4)` 循环，每轮先 `requests.get(current_url)` 并解析出本页 `article.company-card`，
追加进 `companies`；再用 `nav.pagination a:last-child` 取"下一页"的 `href`，
用 `urljoin(current_url, href)` 拼成绝对 URL 赋回 `current_url`，下一轮请求它。
采集满 3 页 `break`；`next_link` 为 `None`（末页）也 `break`，避免死循环。

---

## 4.3 三页采集结果

| 页码 | 企业数 |
|---|---:|
| 1 | 20 |
| 2 | 20 |
| 3 | 20 |
| **合计** | **60** |

- 唯一 `company_id` 数：**60**

---

# 任务 5：结果验证

## 5.1 最终结果

| 检查项 | 结果 |
|---|---:|
| Company 总数 | 60 |
| 唯一 `company_id` | 60 |
| Device 总数 | 5000 |
| 唯一 `device_id` | 5000 |
| Company–Device 不一致企业数 | 0 |

### 【截图 4】最终验证结果

```text
images/04_validation.png
```

```markdown
![最终验证结果](images/04_validation.png)
```

---

## 5.2 一致性检查关键代码

```python
from collections import Counter

device_counts = Counter(
    x["company_id"] for x in devices
)

mismatch_count = 0

for company in companies:
    cid = company["company_id"]
    expected = company["device_count"]
    actual = device_counts.get(cid, 0)

    if expected != actual:
        mismatch_count += 1
```

> 为什么 `status_code == 200` 仍然不能说明"数据已经采集正确"？

答：`200` 仅表示 HTTP 请求-响应未出错，不代表内容正确。服务端异常返回 200 但 body 为空或含错误提示；
参数拼错返回过滤后的子集；只采部分页即中断；页面改版后 selector 匹配到 0 条；字段改名或类型变化取到 `None`
——这些情况 `status_code` 全为 200。因此必须额外做数量校验、ID 唯一性校验、跨数据源一致性校验。

---

# 任务 6：保存采集结果

- `companies.json`：**已生成**
- `devices.json`：**已生成**

---

# 实验结论

静态加载的数据随初始 HTML 返回，用 `requests` 取 `r.text` 后由 BeautifulSoup 配合 CSS Selector 提取；
动态加载的数据不在 HTML 中，需先在 Network 的 Fetch/XHR 中找到 API，直接取结构化 JSON，省去 HTML 解析。
分页方式也不同：HTML 分页依赖 DOM 中的"下一页"链接，取相对 `href` 后用 `urljoin` 拼成绝对地址逐页请求，只能顺序推进；
API 分页直接传 `page` 参数，并可由响应中的 `total` 与 `page_size` 算出总页数，一次覆盖全部数据。
验证上采用三层：数量校验、ID 唯一性校验，以及用 Device 的 `company_id` 计数与 Company 页面 `device_count`
做跨源一致性比对，最终不一致数为 0。
