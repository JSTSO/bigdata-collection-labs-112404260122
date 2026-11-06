# 实验二报告：房源采集与 MySQL 存储

- 班级：数据1241
- 姓名：蒋石涛
- 学号：112404260122
- 实验日期：2026-10-27
- GitHub 仓库链接：https://github.com/JSTSO/bigdata-collection-labs-112404260122
- 源码文件名：`lab02/anjuke.ipynb`（采集，另有同代码的 `anjuke.py`）、`lab02/insert_mysql.py`（入库）
- 数据库与表：`smartpark_lab.houseinfo`
- 数据来源：在线教学站 http://www.bspider.top/anjuke/
- 访问时间及结果（若失败，填写状态码、页面标题或报错）：2026-10-27 19:00 左右，`status_code = 200`，
  页面标题「重庆二手房房产网，重庆二手房出售，重庆买房购房交易信息 - 58安居客」，正文为房源列表，不是验证页。
- 快照文件来源与文件名（在线实验可写“不适用”）：不适用，本次为在线采集

> 填写要求：截图放在 `images/` 中，在对应位置插入图片。代码截图要显示文件名、行号或单元位置，用红框标出关键代码，每项写2—3句话解释。只截最终结果或只贴 AI 生成的文字，不能代替实验过程。未完成的项目写明原因，不填写预期结果，不展示个人账号密码。

---

## 第一阶段：MySQL Docker 更新与启动

- 本次是否需要更新：本次按更新包说明更新（已把 L05 增量更新包中的 `start-mysql.bat`、`mysql-console.bat`、`l05-mysql.ps1`、`l05-mysql-fresh-volume.yaml` 放入原课程环境文件夹，`.env`、`workspace` 与原 `docker-compose.yml` 未覆盖）
- 更新或启动遇到的问题：第一次执行 `start-mysql.bat` 时数据库还在初始化，脚本报健康检查未通过；
  等容器状态变为 `healthy` 后重新执行，显示 `[PASS] L05 MySQL healthy，账号连接检查通过。`

### 启动环境与建表

依次使用 `start-course.bat`、`start-mysql.bat`、`mysql-console.bat` 启动课程环境、数据库和 MySQL 终端。

**截图：** MySQL 启动成功提示，以及终端中 `USE smartpark_lab`、`SHOW TABLES`、`DESCRIBE houseinfo` 的执行结果。可分图。

![MySQL 服务健康状态与账号连接检查](images/00_mysql_ready.png)

上图是 `start-mysql.bat` 的等价检查：容器 `smartpark-student-mysql-1` 状态为 `Up (healthy)`，
用课程账号 `student` 连接 `smartpark_lab` 执行 `SELECT 1` 返回 `1`，说明数据库已就绪、账号可用。

![USE / SHOW TABLES / DESCRIBE houseinfo](images/09_mysql_describe.png)

上图在同一终端里依次执行 `SHOW DATABASES;`（有 `smartpark_lab`）、`USE smartpark_lab;`（`Database changed`）、
`SHOW TABLES;`（有 `houseinfo`）、`DESCRIBE houseinfo;`（9 列，`id` 是 `PRI` + `auto_increment`），
建表语句已生效，字段类型与手册一致。

说明：当前使用哪个数据库？`id` 和8个房源字段分别保存什么？

答：当前使用 **`smartpark_lab`** 库中的 **`houseinfo`** 表。

- `id`：`INT AUTO_INCREMENT PRIMARY KEY`，**自增长主键**，由数据库自动生成，用于唯一标识一行，
  不参与业务比较（自增长 id 不能防止房源重复，判重要按 8 个字段做）。
- 8 个房源字段：`title` 标题（`VARCHAR(200)`）、`house` 楼盘（`VARCHAR(50)`）、`address` 地址（`VARCHAR(200)`）、
  `struct` 户型（`VARCHAR(50)`）、`area` 建筑面积（`VARCHAR(20)`，保留 `㎡` 单位）、
  `floor` 楼层（`VARCHAR(20)`）、`maketime` 建造年代（`VARCHAR(20)`）、`price` 每平方米单价（`VARCHAR(20)`，保留 `元/㎡` 单位）。
  字符集为 `utf8mb4`，保证中文正常存储。

---

## 第二阶段：房源采集与 CSV 数据存储

### 1. 网页分析与房源解析

#### 选择器是怎样得到的

在 Chrome 中按 F12，使用 **Elements → Copy → Copy selector**。选一个字段，记录从完整路径到相对选择器的过程。

- 选择的字段：**标题（title）**
- 复制的原始选择器：

```text
body > div:nth-child(4) > div:nth-child(2) > div:nth-child(1) > div:nth-child(1)
> div:nth-child(1) > div:nth-child(1) > div:nth-child(1) > div:nth-child(1)
> a > div:nth-child(2) > div:nth-child(1) > div:nth-child(1) > h3
```

- 一条房源的父容器选择器：**`div.property`**
- 最终使用的相对选择器：

| 字段 | 选择器（在 `div.property` 内部查找） | 取值方式 |
|---|---|---|
| title | `h3.property-content-title-name` | `get_text(strip=True)` |
| house | `p.property-content-info-comm-name` | `get_text(strip=True)` |
| address | `p.property-content-info-comm-address` | `get_text(strip=True)`（3 个 `span` 一起取） |
| struct | `p.property-content-info-attribute` | `get_text(strip=True)`（6 个 `span` 一起取） |
| area | `p.property-content-info-text` 中文本含 `㎡` 的一项 | 遍历 + 内容判断 |
| floor | `p.property-content-info-text` 中文本含 `层` 的一项 | 遍历 + 内容判断 |
| maketime | `p.property-content-info-text` 中文本含 `年建造` 的一项 | 遍历 + 内容判断 |
| price | `p.property-price-average` | `get_text(strip=True)` |

**截图：** 框出该字段的 HTML 节点和房源父容器，展示选择器的匹配结果。

![房源父容器 div.property 与标题节点的匹配结果](images/06_selector_dom.png)

上图对照了 Copy selector 给出的 13 层位置路径与实际的 `div.property` 卡片结构：
`div.property` 正好包住一整张房源卡（标题、楼盘、地址、属性、价格），`len(soup.select("div.property"))` 返回 **60**。

说明：删掉了哪些层级？怎样确认父容器刚好包含一条房源？

答：Copy selector 生成的路径共 **13 层**，从 `body` 开始全部是 `div:nth-child(n)` 这样的**位置序号**。
删掉的就是这一整串祖先层级——`body`、所有 `div:nth-child(...)`、外层的 `a`，
只保留最后一步的 `h3` 标签，改为**在当前这条房源内部**查找（相对选择器 `row.select_one("h3...")`）。
这样做的好处是：页面在标题上方多插一个广告位或导航条，位置序号会整体错位，而 `div.property` 与 `h3` 的 class 不变。

确认父容器刚好包含一条房源的方法：
① 在 Elements 面板把鼠标移到 `div.property` 上，蓝色高亮区域**恰好框住一整张卡片**，既不只剩半个字段，也没有把第二张卡片包含进来；
② 检查第二、第三条房源，它们用的是同一个 `class="property"`，说明这个 class 是「一条房源」的稳定标志；
③ 在 Python 里跑 `len(soup.select("div.property"))` 得到 **60**，与页面上数出来的 60 条一致，1 个 `div.property` = 1 条房源。

#### 解析8个字段

**截图：** 自己编写的解析代码和首条房源的8个字段输出。

![8 个字段解析代码与首条房源的输出](images/02_parse_fields.png)

上图中红框标出 `soup.select("div.property")`（取整页 60 条）与 `row.select_one(...)`（在一条房源内取单个字段），
以及面积 `㎡`、楼层「层」、年代「年建造」三处**按内容判断**的取值，和单价 `property-price-average` 的取值；
下方输出是首条房源的 8 个字段。

说明：两个选择方法各用在哪里？怎样确认面积和单价没有取错？

答：`select()` 用于**取多条**——`soup.select("div.property")` 一次拿到整页 60 条房源，
`info.select("p.property-content-info-text")` 拿到一条房源里的全部属性文本（后面要遍历它挑面积、楼层、年代）。
`select_one()` 用于**取一条**——8 个字段各自只需要第一个匹配节点，如 `row.select_one("h3.property-content-title-name")`。

确认面积和单价没取错，做法是**先打印属性区每一项、再按内容特征取值**：

1. **面积**：属性区有 5 个 `p.property-content-info-text`，依次是户型、面积、朝向、楼层、年代。
   打印后看到第 2 项是 `89.9㎡`，与网页一致。代码用「文本含 `㎡`」判断而不是用下标 `[1]`，同时保留单位。
2. **楼层**：判断是否含「层」字，取到 `低层(共27层)`，**没有**误取成第 3 项的朝向 `南北`。
3. **单价**：价格区有两个节点——`p.property-price-total` 是总价 `148 万`，`p.property-price-average` 才是单价 `16465元/㎡`。
   程序取后者，单位是元/㎡，不是万元总价。
4. **建造年代**：判断是否含「年建造」，取到 `2019年建造`；缺失时保存空字符串。

另外三条样本（第 1 条、跨页第 61 条、末条第 600 条）的面积 `89.9㎡ / 85.7㎡ / 90.2㎡`、
单价 `16465 / 16914 / 15530 元/㎡` 都与网页逐字一致。

### 2. 下一页采集

#### “下一页”节点的真实 HTML

在页面底部「下一页」上右键 → 检查，第 1 页得到的原始标签是：

```html
<a href="?page=2" data-npv="" class="next  next-active" data-v-29d65fc6="">下一页</a>
```

| 部分 | 值 | 说明 |
|---|---|---|
| `href` | `?page=2` | **要取的东西**，相对地址，需 `urljoin` 补全 |
| `class` | `next  next-active` | **要用的东西**：`next` = 这是下一页按钮，`next-active` = 按钮当前可用 |
| `data-npv` | 空 | 站点统计属性，与采集无关 |
| `data-v-29d65fc6` | 空 | Vue 的 scoped CSS 哈希，**每次编译都会变**，不能写进 selector |

**末页（第 10 页）的同一个按钮**变成：

```html
<a href="#" data-npv="" class="next " data-v-29d65fc6="">下一页</a>
```

标签还在、`a.next` 依然能匹配到，但 `next-active` 没了、`href` 变成 `#`。
所以程序用 `a.next.next-active` 定位：末页时匹配结果为 `None`，据此判定到达末页。
若只用 `a.next`，末页也会命中，就会拿着 `href="#"` 一直翻下去。

- 下一页 `href` 原值：`?page=2`
- 用 `urljoin()` 补全后的地址：`http://www.bspider.top/anjuke/?page=2`
- 实际停止原因：**明确禁用的末页按钮**——第 10 页「下一页」只剩 `class="next"`、`href="#"`，
  `select_one("a.next.next-active")` 返回 `None`，程序 `break` 并打印「已到末页」。

**截图：** 框出提取 `href`、补全地址、循环更新地址和停止条件的代码。附前两页的运行日志，以及最终停止提示。

![下一页 href、urljoin、循环更新地址与停止条件](images/03_next_page.png)

上图中红框依次是：`next_a = soup.select_one("a.next.next-active")`、
`href = next_a.get("href")`、`next_url = urljoin(current_url, href)`、
`current_url = next_url` 与末页 `break` 条件；下方是第 1、2 页的翻页日志和最终停止提示。

说明：怎样进入下一页？怎样防止重复采集或无限循环？若中途失败，说明停在哪一步。

答：

- **进入下一页**：每轮先 `get_soup(current_url)` 请求并解析本页 60 条，追加进 `out_list`；
  再 `next_a = soup.select_one("a.next.next-active")`，`href = next_a.get("href")` 得到 `?page=2`，
  用 `urljoin(current_url, href)` 补成 `http://www.bspider.top/anjuke/?page=2`，
  赋回 `current_url` 再进下一轮。
- **防止重复采集**：`seen_urls` 集合记录已请求过的地址，循环开头 `if current_url in seen_urls: break`，
  同一地址不会被请求第二次；日志里每页的地址都不同。
- **防止无限循环**：四道闸——① `page_no <= MAX_PAGES`（上限 30 页）；
  ② 找不到 `a.next.next-active` 就停；③ `href` 为空或等于 `#` 就停；
  ④ `next_url.startswith(PREFIX_OK)` 校验地址没有越出教学站。页间 `time.sleep(1.2)`，至少等 1 秒。
- **是否中途失败**：没有失败。日志中有 4 次 `ReadTimeoutError`（第 2、3、8、10 页），
  都是单页请求超时，`get_soup()` 内部重试后成功，没有跳页，10 页全部采到。

### 3. 保存 CSV

**截图：** CSV保存与回读代码、回读条数。框出编码、表头和数据写入部分。

![CSV 保存与回读核对](images/04_csv_save.png)

上图红框标出 `encoding="utf-8-sig"`、`newline=""`、`writer.writerow(FIELDS)` 表头写入、
`writerows` 数据写入，以及 `csv.DictReader` 回读并统计行数；下方输出是回读结果。

![CSV 回读条数：数据行 600 行，含表头 601 行](images/05_csv_count.png)

回读结果：**数据行 600 行**（表头不计入），文件总行数 601 行，与采集条数 10 × 60 = 600 一致。

说明：8个字段的顺序是什么？按8个字段完全相同去重后，保存了多少条记录？

答：字段顺序为 `title → house → address → struct → area → floor → maketime → price`。
表头和数据都由同一个 `FIELDS` 列表生成，保证顺序一致、每条记录恰好 8 项。
文件用 `encoding="utf-8-sig"`（带 BOM，Excel 打开中文不乱码）、`newline=""`（避免 Windows 下多出空行）。

**去重情况**：程序按 8 个字段完全相同统计到重复记录 **102** 条。
本次 `houseinfo.csv` 保存 **600 条**数据行（另加 1 行表头，共 601 行），即**保留全部记录、未做去重**；
去重后的 498 条在第三阶段入库时完成（见下文），因此数据库最终是 498 条。

重复来源已核查：另外单独请求第 2、3、10 页，与已保存的 HTML 逐条比对，三页内容完全一致，
说明页面是确定的、不是随机抽样；代码也没有重复请求同一地址（`seen_urls` 拦截）。
所以这 102 条重复是**教学站数据本身**在页内和跨页重复出现（同一房源重复挂出，8 个字段却完全一致），
不是分页循环造成的重复采集。

#### CSV 开头（表头 + 前 5 条数据）

![CSV 表头与前 5 条数据](images/01_csv_head.png)

```csv
title,house,address,struct,area,floor,maketime,price
万科物业自营 南坪六公里金域翰林 三号线精装正三房带家具电器,万科金域学府翰林菁华里,南岸学府大道二塘路59号,3室2厅1卫,89.9㎡,低层(共27层),2019年建造,16465元/㎡
黄泥磅 两代一家 正规三房双卫 业主自住房 拎包入住,两代一家,渝北渝北黄泥磅紫荆路29号,3室2厅2卫,94.5㎡,低层(共28层),2008年建造,14815元/㎡
鸳鸯16年次新小区 大雅云居山刚需经典3房户型 随时看房诚售,大雅云居山,渝北鸳鸯龙展路59号,3室2厅1卫,88㎡,低层(共24层),2016年建造,16487元/㎡
中交中央公园 横厅精装大两房 出门都是公园,中交中央公园悦颂,渝北中央公园翠丹路4号,2室2厅1卫,91㎡,低层(共25层),2016年建造,20660元/㎡
富力现代广场 豪装大两房 户型方正 看房方便,富力现代广场,南岸弹子石弹子石新街59号,2室2厅1卫,85.5㎡,低层(共30层),2012年建造,16969元/㎡
```

---

## 第三阶段：MySQL 数据存储

### 1. 读取 CSV 并入库

本阶段读取第二阶段的 `data/houseinfo.csv`，不重新采集。

**截图：** 框出读取CSV、按8个字段整理入库参数的代码。

![读取 CSV 并按 8 个字段整理入库参数](images/07_insert_read_csv.png)

上图红框是 `FIELDS` 固定顺序（CSV 表头、INSERT 字段、参数顺序共用同一个列表）、
`csv.DictReader(f)` 读取、表头不一致时 `raise ValueError`、以及
`return [tuple(r[k] for k in FIELDS) for r in rows]` 把每条记录整理成 8 元组。输出为「读入 600 条，每条 8 个字段」。

**截图：** 自己编写的入库代码。框出 `executemany()`、`commit()`、`rollback()`，以及关闭游标和连接的操作。另框出避免重复插入的代码。

![批量入库：判重、executemany、commit、rollback、关闭](images/08_insert_executemany.png)

上图红框共五处：
① `SELECT` 读出库内已有 8 字段记录存进 `existed` 集合（判重依据）；
② `seen = set(existed)` 后逐条 `if rec in seen: continue`，跳过库内已有记录和同批次内重复，
得到 `to_insert`；
③ `affected = cur.executemany(INSERT_SQL, to_insert)` 批量写入、
`conn.commit()` 提交；
④ `except` 分支 `conn.rollback()` 回滚，保证半批数据不留在库里；
⑤ `finally` 中 `cur.close()` / `conn.close()` 释放资源（先判断不为 `None`）。
下方是第 1 次运行的真实输出：入库前 0 条 → 待插入 498、跳过 102 → 提交后 498 条。

说明：每条记录怎样对应8个参数？成功、失败时分别做什么？再次运行时怎样避免重复插入？

答：

- **8 个字段 ↔ 8 个参数**：`INSERT INTO houseinfo (title, house, address, struct, area, floor, maketime, price) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)`。
  括号里的字段顺序与 `FIELDS` 列表、CSV 表头顺序完全一致；每条记录由
  `tuple(r[k] for k in FIELDS)` 生成为 8 元组，`executemany` 把 498 个这样的元组依次填进 8 个 `%s`，
  值的类型和文字（含 `㎡`、`元/㎡` 单位）原样带入，不做二次修改。SQL 用参数占位，不用字符串拼接或 f-string，避免转义问题和注入。
- **成功时**：`executemany` 返回影响行数 → `conn.commit()` 提交 → 再 `SELECT COUNT(*)` 复查条数，确认真的落库，
  最后 `finally` 关闭游标和连接。返回值是本次影响行数，不是库里总条数，所以另外查了总数。
- **失败时**：`except` 里 `conn.rollback()` 撤销本次事务中尚未提交的修改，把异常 `raise` 抛出（不吞错），
  再由 `finally` 关闭游标和连接。回滚只能撤销未提交的修改，不能撤销已经提交的数据，也不能撤销建表。
- **避免重复插入**：自增长 `id` 挡不住重复，所以先把库内已有的 8 字段记录读成集合 `existed`，
  逐条比较；已存在就 `continue`，只把库里没有的记录交给 `executemany`。同一次读取中出现的重复，
  用 `seen.add(rec)` 也一并挡掉。**没有清空表重跑**——重跑只是「新增 0」。
  第 2 次运行的真实输出见下图。

![第 1 次运行与关闭程序后第 2 次运行](images/12_insert_rerun.png)

上图下半部分是关闭程序后对同一批 CSV 的独立重跑：入库前已有 498 条，待插入 **0** 条、跳过 600 条，
打印「库内已是最新，本次不写入任何数据」，当前条数仍是 498，没有重复增加。

### 2. 核对结果

#### 查询条数和样本

**截图：** MySQL 终端中的入库前后 `SELECT COUNT(*)`、重跑后的条数，以及 `SELECT ... LIMIT 5` 的结果。

![入库后条数与 8 字段重复性检查](images/10_mysql_count.png)

入库后 `after_count = 498`；再用 `COUNT(DISTINCT CONCAT_WS('|', 8 个字段))` 检查，
`total = 498`、`uniq_rows = 498`，说明库里 498 行**没有 8 字段完全相同的重复行**。
（入库前 `before_count` 为 **0**，见 `08_insert_executemany.png` 输出里的「入库前条数：0」。）

![入库后 SELECT ... ORDER BY id LIMIT 5 的样本](images/11_mysql_select5.png)

上图用 `\G` 竖排输出前 5 条，8 个字段完整、中文正常、面积与单价都带单位。

| 项目 | 实际结果 |
|---|---|
| 成功采集页数 | 10（第 1～10 页，每页 60 条） |
| 采集记录数（去重前） | 600 |
| 完全相同的重复记录数 | 102 |
| CSV数据行数（不含表头） | 600（保留全部，未去重）；去重后 498 |
| 入库前条数 | 0 |
| 入库后条数 / 本次新增条数 | 498 / 498 |
| 关闭上次程序后，同批数据独立重跑的条数 / 新增条数 | 498 / 0 |
| 是否采集完整；未完成的原因 | 完整。10 页全部采到，第 10 页「下一页」按钮已禁用（`href="#"`，无 `next-active`），属正常结束 |

说明：首次入库后的条数应等于去重后的CSV数据行数，8个字段的内容也应一致。若不一致，说明原因。总条数以实际结果为准。

答：一致。**首次入库后 498 条 = 去重后的 CSV 数据行数 498**，`executemany` 影响行数也是 498。
差 102 条正是 CSV 里 8 个字段完全相同的重复记录，在入库阶段被判重跳过（`待插入：498 条，跳过重复：102 条`）。
8 个字段内容一致：读取时按 `FIELDS` 顺序取元组、写入时字段顺序与 `%s` 位置一一对应、
连接与表都用 `utf8mb4`，没有任何转换或截断，中文、`㎡`、`元/㎡` 原样保存。
重跑一次新增 0，`total = uniq_rows = 498`，说明既没漏插也没重复插。
总条数以实际结果为准：本次是 600 条采集 / 498 条去重后入库，没有追求旧报告的 601 条。

#### 逐条核对

选取第一条、跨页一条和最后一条，比较网页、CSV与数据库中的8个字段。

**截图：** 三条样本的对照结果，框出面积、单价和年代。可分图。

![三条样本：网页 = CSV = 数据库 逐字段对照](images/13_sample_compare.png)

| 字段 | 样本1 | 样本2 | 样本3 |
|---|---|---|---|
| title | 万科物业自营 南坪六公里金域翰林 三号线精装正三房带家具电器 | 万科物业自营 朝中庭 业主诚心出售 | 恒大照母山 精装3房 内圈位置 户型方正 拎包即住 诚意出售 |
| house | 万科金域学府翰林菁华里 | 万科金域学府翰林菁华里 | 恒大照母山山水城 |
| address | 南岸学府大道二塘路59号 | 南岸学府大道二塘路59号 | 渝北照母山恒康路9号 |
| struct | 3室2厅1卫 | 3室2厅1卫 | 3室2厅1卫 |
| **area** | **89.9㎡** | **85.7㎡** | **90.2㎡** |
| floor | 低层(共27层) | 低层(共27层) | 低层(共32层) |
| **maketime** | **2019年建造** | **2018年建造** | **2015年建造** |
| **price** | **16465元/㎡** | **16914元/㎡** | **15530元/㎡** |

- 样本1（页码或URL、记录位置）：第 1 页 `http://www.bspider.top/anjuke/` 第 1 条（整页第 1 条，`id = 1`）
- 样本2（页码或URL、记录位置）：第 2 页 `http://www.bspider.top/anjuke/?page=2` 第 1 条（整页第 61 条）
- 样本3（页码或URL、记录位置）：第 10 页 `http://www.bspider.top/anjuke/?page=10` 最后一条（整页第 600 条）

核对结论及差异：三条的 8 个字段在**网页（保存的 HTML 快照）、CSV、数据库**三处逐字相同，无差异。
重点核对项：面积均带 `㎡` 单位（89.9 / 85.7 / 90.2）；单价是**每平方米单价**（16465 / 16914 / 15530 元/㎡），
不是 `148 万` 这类总价；建造年代均为 `20XX年建造` 且未缺失；地址三个 `span` 拼接完整、没有被插入多余空格；
楼层是 `低层(共27层)` 这类，未误取朝向 `南北`。

---

## 问题与解决

- 遇到的问题、处理方法及结果：

  1. **只用 `User-Agent: Mozilla/5.0` 请求会返回「访问验证」页**：`status_code` 仍是 200，
     但标题是「访问验证-安居客」，`div.property` 匹配到 **0** 条。
     处理：改用完整的浏览器请求头（`Accept`、`Accept-Language`、`Accept-Encoding`、`Referer` 等），之后正常返回 60 条。
     这是「200 不代表内容正确」的实例，所以每页都额外打印了本页条数。
  2. **请求间歇性 `ReadTimeoutError`**：教学站响应慢，出现过约 50 秒才返回的情况。
     处理：改用 `timeout=(10, 60)`，并在 `get_soup()` 内最多重试 5 次、每次等待 5 秒。
     日志里 4 次失败都重试成功，没有丢页。
  3. **楼层和朝向容易混淆**：属性区第 3 项是朝向（`南北`）、第 4 项才是楼层。
     处理：不按下标取值，改按内容特征（含「层」字）判断。
  4. **入库时表里已有旧数据会重复插入**：处理见第三阶段——先读出库内 8 字段集合再判重，不清表、不靠自增 id。
  5. **终端中文显示为 `?`**：MySQL 客户端默认字符集不是 utf8mb4。
     处理：连接客户端时加 `--default-character-set=utf8mb4`，并在建表时指定 `CHARACTER SET utf8mb4`。

- 尚未解决的问题、未完成的步骤：无。三个阶段均已完成；第三阶段重跑验证新增 0 条。

## 思考题

1. 本次怎样实现下一页采集？结合自己的代码说明。

   答：`crawl()` 用 `while page_no <= MAX_PAGES` 循环。每轮 `get_soup(current_url)` 请求并解析，
   `soup.select("div.property")` 取本页 60 条追加进 `out_list`；然后
   `next_a = soup.select_one("a.next.next-active")` 定位**活动的**下一页按钮（末页时该按钮没有 `next-active`，直接返回 `None`），
   `href = next_a.get("href")` 得到 `?page=2`，`urljoin(current_url, href)` 补成
   `http://www.bspider.top/anjuke/?page=2`，赋回 `current_url` 后进入下一轮。
   停止条件有三类：找不到 `a.next.next-active`（末页按钮已禁用）、`href` 为空或为 `#`、
   地址已访问过或越出教学站前缀；另有 `MAX_PAGES` 防死循环。本次在第 10 页因按钮禁用正常结束。

2. 旧代码把 out_list 定义在全局。你把结果列表放在哪里？为什么？

   答：放在 `crawl()` 函数内部（`out_list = []`），用 `return out_list, page_no` 返回给调用方。
   这样每次调用都是全新的空列表，函数重复调用不会累积上次的数据；也消除了修改全局变量的副作用，
   程序更好测试和复用。`seen_urls` 同样放在函数内，保证每次采集的「已访问地址」记录是干净的。

3. 你怎样获取建造年代？节点缺失或顺序改变时，怎样处理？

   答：建造年代在 `p.property-content-info-text` 节点里，文本形如 `2019年建造`。
   程序不用「第 5 个 p」这种下标，而是遍历属性区所有 `property-content-info-text`，
   用 `if not maketime and "年建造" in t` 判断，只有含「年建造」才认作年代。
   这样页面少一项或顺序被打乱，也不会取到面积或楼层。`maketime` 初值为空字符串，
   `if not maketime` 既做「还没取到」的标记，也保证节点缺失时不抛异常，字段保存为空，符合手册要求。

## 提交检查

- [ ] 实验一原仓库的 `lab02/` 中已有源码、CSV、报告和真实截图。
- [x] 截图清楚，关键代码已框出，每项附有个人解释。
- [ ] GitHub上能打开文件和图片，未上传个人账号密码、`.env`或教师答案。
- [x] 未完成项已如实说明；学习通只交原仓库链接。

如使用 AI，简述它帮助了什么，以及自己修改、运行和核对了什么：

本次使用 AI 辅助编写解析函数、分页循环和入库程序的初稿，并生成代码截图。
本人完成了：在 Chrome 中核对 `div.property` 与 8 个字段的 DOM 结构、记录「下一页」的真实标签；
解决访问验证页与超时重试问题；运行采集脚本并逐页查看翻页日志（10 页 600 条）；
回读 CSV 核对 600 行与表头；在 MySQL 终端执行 `DESCRIBE`、`COUNT(*)`、`LIMIT 5` 验收入库结果；
关闭程序后重跑一次确认新增 0 条；抽取第一条、跨页第 61 条、末条第 600 条做网页 / CSV / 数据库三方逐字段核对。
