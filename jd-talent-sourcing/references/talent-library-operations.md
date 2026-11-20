# 人才库操作细节

## 适用范围与运行配置

使用 `browser` skill 完成使用者授权招聘平台的交互。本文件仅保存通用操作方法，不包含任何企业专属域名、路由、接口、字段编码、人员、岗位、资源绑定或既往执行记录。

运行前从使用者授权配置取得平台入口、详情入口及可见控件语义；通过页面核实实际路径、过滤参数名、合法字段和值、分页及状态展示方式。没有配置或无法核实时，仅补问对应缺口，不猜企业地址或参数编码。

登录、验证码、权限及工具连接是前置条件。不要直接请求招聘接口，不读取凭证，不引入其他自动化技术栈或复制依赖工具的实现。新增复杂读取方案须先解释并取得授权。

## 标签控件含义

- 学历多选与隐藏升学路径等选项分别核实；页面学历标签不证明学校性质或第一学历。
- 工作年限的枚举、自定义范围和实习计算口径分别确认；实际上下界来自岗位配置，不预设具体数值。
- 标签运算符与选项分别读取，不丢失“包含／不包含”等语义。未知选项先读页面，不猜内部ID。
- 综合评价过滤不代替详情面评核验，也不自行添加为硬门槛。等级顺序、数值映射和历史范围使用本岗位确认配置。
- 锁定过滤与详情当前状态分别验证。状态区域完整加载且其显示语义经确认后，才可依据锁定标志的有无判断；单纯文字缺失不证明未锁。
- 隐藏已读、地点、排序及其他非硬性条件均回读选中态；仅打开选项不等于选择生效。

## 标签操作闭环

1. 保存当前筛选快照，从当前筛选组文字打开，避开关闭图标。重新核实DOM，不保存一次性序号或定位引用。
2. 每次打开重读页面，等待可见弹层及选项；readyState不证明动画或数据加载完成。
3. 核查可见性、位置、遮挡与选中态；隐藏、零尺寸、屏外节点不证明控件失效，必要时滚动正确容器。
4. 选择已确认值，核实是否需确认或应用后关闭；不推断所有控件即时生效。
5. 回读标签文字、运算符、值及URL，并验证结果加载。请求发出、Done、URL变化或人数不变不单独证明过滤成功。
6. 快照、DOM与截图矛盾时记录差异，核对当前可见页面；有限合理路径仍无法验证即停止受影响操作。

## 参数化URL恢复

只有目标平台确实以URL中的JSON参数表达筛选、参数结构已核实且使用者授权恢复条件时，才使用以下导航逻辑。它不请求接口，也不读取登录信息。保留其他查询参数与排序，仅修改已核实的筛选键；执行前保存原快照，执行后逐项回读，失败恢复原配置。

通过 `browser` skill 当前支持的页面执行能力使用此代码。`options`的所有值来自本轮授权配置：

- `expectedOrigin`：实际核实的平台origin。
- `expectedPath`：实际核实的列表pathname。
- `filtersParam`：实际核实的JSON筛选查询参数名。
- `allowedKeys`：本次已获授权且结构已核实的筛选键白名单。
- `patch`：本轮确认的合法筛选值对象；不能包含凭证或其他无关数据。

```javascript
function restoreVerifiedFilters(options) {
  const isRecord = value =>
    value !== null && typeof value === 'object' && !Array.isArray(value);
  if (!isRecord(options) ||
      typeof options.expectedOrigin !== 'string' || !options.expectedOrigin ||
      typeof options.expectedPath !== 'string' || !options.expectedPath ||
      typeof options.filtersParam !== 'string' || !options.filtersParam ||
      !Array.isArray(options.allowedKeys) ||
      !options.allowedKeys.every(key => typeof key === 'string' && key.length > 0) ||
      !isRecord(options.patch)) {
    return {ok: false, reason: 'invalid_config'};
  }
  if (location.origin !== options.expectedOrigin ||
      location.pathname !== options.expectedPath) {
    return {ok: false, reason: 'wrong_page'};
  }
  const url = new URL(location.href);
  let filters;
  try {
    filters = JSON.parse(url.searchParams.get(options.filtersParam) || '{}');
  } catch (_) {
    return {ok: false, reason: 'invalid_existing_filters'};
  }
  if (!isRecord(filters)) {
    return {ok: false, reason: 'invalid_existing_filters'};
  }
  const entries = Object.entries(options.patch);
  const allowed = new Set(options.allowedKeys);
  const blocked = new Set(['__proto__', 'prototype', 'constructor']);
  if (entries.length === 0 ||
      entries.some(([key]) => !allowed.has(key) || blocked.has(key))) {
    return {ok: false, reason: 'unapproved_patch'};
  }
  try {
    for (const [key, value] of entries) filters[key] = value;
    url.searchParams.set(options.filtersParam, JSON.stringify(filters));
    location.assign(url.href);
    return {
      ok: true,
      action: 'navigation_requested',
      changedKeys: entries.map(([key]) => key)
    };
  } catch (_) {
    return {ok: false, reason: 'navigation_failed'};
  }
}
```

调用时向函数传入真实已核实的配置对象；不在技能中保存该对象。`navigation_requested`仅表示请求导航，不代表筛选验收成功。该方法不适用于未核实的URL结构，不能用它绕过控件失败、登录、验证码或权限。

## 详情材料读取

进入详情记录稳定标识、当前投递关系与材料状态；同人不同投递不重复计数。打开详情可能改变网站已读状态，记录真实行为，不额外点击“标已读”。取得明确硬淘汰证据可短路并记录未查项；个体缺项不自动视为整批阻塞。

### 已有备注

1. 从可见备注入口打开抽屉，区分已有记录与上方新增／编辑区；不切换意向、私密、翻译或提交。
2. 读取已确认范围内的正文、作者、真实时间及关联职位，滚动列表补齐覆盖。空编辑框、未加载或读取失败不等于无备注；只有数量、列表及明确无记录状态相互一致时，才能判断没有记录。
3. 使用经核实的关闭按钮退出，避开取消／确定／提交；键盘关闭是否生效需回读，Done不算关闭证据。

### 历史投递

1. 展开投递列表，读取总数及分页／滚动覆盖；选择相关投递后回读职位和流程确实切换，不执行通知、转移、恢复或加入职位。
2. 展开相关历史卡片，读取投递日期、流程状态、终止阶段及原因；未展开内容标未读，终止状态不等于面试不通过。
3. 投递数量不等于有效面评数量。若岗位要求全部历史最近若干次，先补齐全历史范围，不提前筛掉其他职位或负责人。
4. 分清主列表、详情内部及投递列表的分页与滚动，不能以一处覆盖代替其他区域。

### 面评与时间排序

1. 依当前控件展开评价内容、结论及等级，回读完整材料，不按固定图标序号点击。
2. 关联人员、投递、实际面试时间及提交时间；按岗位明确要求选用排序时间。附近日期的语义不明时待核验，不互相替代。
3. 按本岗位确认的历史范围补齐已完成面评，使用确认的时间字段排序，再检查最近若干次、负责人、专业与评分等组合条件；本文件不预置具体数量或等级规则。
4. 无评分、未完成、未读到和低评分分别记录；数值映射只采用已确认口径。未推进、终止或综合分数不代替具体淘汰条件。

## 操作验收

按目标平台逐项验收备注开／读／关、投递开／切换、面评展开及时间字段。区分“观察到”“自动操作回读通过”“未验证”。只读切换尽量返回原投递，不修改人员或流程。

标签设置成功、学会页面操作和寻访目标完成是不同状态。持续阻塞时报告具体字段、已读／未读范围及恢复位置；正式完成按主文计数与写入回读门槛判定。所有真实配置、证据与验收记录仅保存在授权运行资源中，不写回本包。
