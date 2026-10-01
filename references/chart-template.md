# 笔吧风格价格仪表盘：模板改造手册

模板：本技能自带 `assets\笔记本价格走势仪表盘.html`（安装位置 `C:\Users\hubo\.agents\skills\price-history-analysis\assets\`，同目录已带 `echarts.min.js`；工作区副本在 `C:\Users\hubo\Desktop\agent_workspace\新建文件夹\`）。
依赖：与 HTML 同目录的 `echarts.min.js`（5.5.x；缺失时离线重下：
`curl -o echarts.min.js https://registry.npmmirror.com/echarts/5.5.1/files/dist/echarts.min.js`）。

## 改造五步

1. **复制模板**到目标工作区（连同 echarts.min.js）。
2. **改 `MODELS` 数组**——每个机型一个 `M(...)`：

```js
M(id, chip, rest, cfg, sku, status, extra, anchors, gaps)
// chip  = 选择器里红色高亮的短名（型号关键字）
// rest  = 配置摘要 + 条数（展示用）
// cfg   = 完整配置行；sku = SKU号；status = '在售' | '无货' | '已下架'
// extra = 统计行文字（紫色补充X条 · 邻近推定Y条 · 国补存疑Z条 …）
// anchors = [ ['YYYY-MM-DD', 价格, 'adopt|calc|tip|weak|oos', '备注（含来源）'], … ]（时间升序）
// gaps  = [ ['起','止'], … ] 无数据空窗，画"缺价格数据"虚线框
```

   锚点类型与图形映射（写死在 KIND_COLOR/KIND_NAME，别改语义）：
   `adopt`=蓝实心圆（直接采信）、`calc`=橙三角（补算国补后）、`tip`=紫三角（爆料凭证）、
   `weak`=灰空心圆（邻近推定/不可信）、`oos`=深灰圆（无货定格，末次价）。

3. **改 `IP_ROWS`**（暗色区"整数价月度占比"表；没有口径渗透率故事就整块删掉 `.table-wrap`）：
   `[月份, 有20%台阶机型数, 有15%台阶机型数, '占比%', 是否红框行, 是否加粗]`。
4. **改结论卡 `verdicts`（4 张暗色卡）与页头副标题**：涨幅结论、数据截止日、
   重建数据免责声明（若曲线是重建演示必须写明）。
5. **验证**：
   - 提取 `<script>` 用 `node --check` 查语法（写错一个引号全页静默变白）；
   - Edge 无头整页截图：`msedge --headless=new --disable-gpu --screenshot=out.png --window-size=1280,2600 --virtual-time-budget=6000 "file:///…html"`，
     逐项确认：选择器有字、主图渲染、覆盖条在、标注卡不重叠不出界、暗色区表格有行。

## 已踩过的坑（照做免重查）

- **custom 系列在 time 轴上 `api.size()` 返回 NaN** → 覆盖条静默不渲染。日格宽度必须用相邻两天 coord 差：`api.coord([v+86400000,0.5])[0] − api.coord([v,0.5])[0]`。
- **markLine 竖线 label 用 `position:'end'`**（顶外侧）；`insideEndTop` 会把文字挤成竖排。两条线用不同 `distance` 防重叠。
- **标注卡（markPoint roundRect + rich 白底）同屏 ≤2 张**：模板策略=最早一条 + 居中一条，`dx` 按锚点在时间轴的分数位置选左/右（<0.5 向右 +155，否则 −155），`dy` 按价格是否 ≥ 中位数选上/下（+84 / −88）。多了必重叠。
- **grid.left ≥ 82**，否则 Y 轴千分位标签和最左 X 轴标签被裁。
- 时间轴 `axisLabel.hideOverlap:true`，否则月初/月中 tick 重复显示同月。
- 事件卡 `coord` 必须传时间戳 `ts(a[0])`，字符串日期在 markPoint 上不稳。
- 对比模式：comp 系列统一 A 蓝 / B 橙；`legendSingle`/`legendComp` 两个图例 div 的 display 切换在 `renderMeta()` 末尾，新增图例记得同步。

## 页面功能与 URL 参数

- 三个下拉：机型与配置（红色高亮型号）、对比机型（选后切 A/B 对比模式）、时间轴长度（全宽/近12/近6/近3月）。
- URL 直达：`?m=<机型id>&c=<对比机型id>&r=full|m12|m6|m3`（例 `?m=tx7p&c=tx6p`）。
- 悬浮 tooltip：日期 + 价格 + 来源类型 + 备注——锚点 `anchors[i][3]` 的备注写来源，报告里就不必重复。
- 覆盖条颜色语义（`dayColor`）：蓝=当天有采信记录、绿=仅补算/爆料、浅灰=无记录、深灰=无货/不可信。

## 结论表述规范（配图输出时）

- 涨幅一律"时段最低价(日期) → 末次价(日期) = +X%"，两代小改款要连起来算"接力涨幅"。
- 无货/下架机型必须写"已定格/买不到"，推定占比高要写"曲线可信度中等"。
- 历史低点规律只陈述曲线可见事实（如"双11 是全年最低点"），不做无依据预测。
