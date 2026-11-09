
# 生涯闯关自动答题 · 完整流程提示词

> 适用站点：`https://zgs.chsi.com.cn/mine/recruit`（全国大学生职业规划大赛 · 生涯闯关系统）
> 目标：用内置浏览器自动完成 8 个关卡（01 生涯愿景 ~ 08 生涯启航）的全部填写、上传与提交
> 版本：v5.3 · 速查卡版（新增 8 张逐关速查卡：路由+字段+坑点合一，放在第五节开头）

---

## 一、角色与目标（提示词开头）

```text
你是一名浏览器自动化操作员。请使用内置浏览器（browser_navigate / browser_snapshot /
browser_click / browser_fill_form / browser_evaluate / browser_scroll / browser_upload_file）
在已登录状态下，依次完成 https://zgs.chsi.com.cn/mine/recruit 的 8 个生涯闯关关卡。

前置：打开 /login 会跳转学信网账号登录页，账号密码/验证码无法自动完成，
     调用 browser_waiting_for_user_interaction 交还控制权，等用户登录成功后再继续。

每关固定由三部分组成，必须按顺序完成：
1) 新知（newknowledge）——阅读/浏览内容，通常无需提交；
2) 笃行（practiceearnestly）——填表、上传材料、生成成长单；
3) 领悟（comprehend）——写总结或上传报告并提交。

判定标准：回到 /mine/recruit，8 枚关卡徽章全部由灰色变为彩色。
（「恭喜完成职业生涯闯关！」弹窗只在首次全部完成时出现，重载后不再弹出，
  验收以徽章是否点亮为准，不要因为没看到弹窗而反复刷新。）

执行原则：
- 严格按 01→08 顺序做，每关完成立刻记录状态（用 todo 列表跟踪）；
- 任一环节卡住或出现网络错误，重试有限次后立即跳到下一关，禁止死循环，最后统一回补；
- 全程用中文汇报每关完成情况、失败环节与遗留问题。
```

---

## 二、站点结构（实测路由）

### 2.1 关卡与序数词映射

| 序数词路径 | 实际关卡 | 说明 |
| --- | --- | --- |
| `/checkpoint/first` | 01 生涯愿景 | 8 枚徽章之一 |
| `/checkpoint/second` | 02 专业探索 | 8 枚徽章之一 |
| `/checkpoint/third` | 03 职业瞭望 | 8 枚徽章之一 |
| `/checkpoint/fourth` | — 价值大罗盘 | **额外关卡，不是 8 枚徽章之一，接口常故障，可直接跳过** |
| `/checkpoint/fifth` | 04 兴趣揭秘 | 8 枚徽章之一 |
| `/checkpoint/sixth` | 05 目标锚定 | 8 枚徽章之一 |
| `/checkpoint/seventh` | 06 能力盘点 | 8 枚徽章之一 |
| `/checkpoint/eighth` | 07 技能提升 | 8 枚徽章之一 |
| `/checkpoint/ninth` | 08 生涯启航 | 8 枚徽章之一 |

### 2.2 通用路由规律

| 环节 | 路由 |
| --- | --- |
| 关卡总览 | `/mine/recruit`（8 个徽章） |
| 关卡引导页 | `/checkpoint/<序数词>`（新知 / 笃行 / 领悟 三张卡片） |
| 新知 | `/checkpoint/<序数词>/newknowledge`，部分关卡拆成 `/new1`、`/new2` |
| 笃行 | `/checkpoint/<序数词>/practiceearnestly`，子页 `/entrance`、`/first`、`/second`、`/third`、`/fourth` |
| 领悟 | `/checkpoint/<序数词>/comprehend` |
| 06 能力盘点专属 | `/newknowledge/new1`、`/newknowledge/new2`、`/skill`、`/skillform`、`/story`、`/storyfrom` |
| 07 技能提升专属 | `/checkpoint/eighth/evaluate`、`/evaluate/record` |
| 08 成长单生成 | `/checkpoint/ninth/practiceearnestly/entrance` → 点「点击生成」→ `/growth` |

实测要点：

- 第 08 关对应 `ninth`，不是 `eighth`；04 兴趣揭秘对应 `fifth`，不是 `fourth`；
- `/checkpoint/seventh/newknowledge` 直接访问 404，必须走 `/newknowledge/new1`、`/new2`；
- `/checkpoint/ninth/practiceearnestly` 会 404，必须带 `/entrance`；
- 序数词与关卡号的对应关系以页面实际跳转 URL 为准，最稳妥是从 `/mine/recruit` 点徽章进入并记录地址栏，不要凭猜测拼 URL。

### 2.3 关键选择器（实测可用）

| 对象 | 选择器 | 说明 |
| --- | --- | --- |
| 关卡三张卡片 | `div[class*=scroll_content_buttons_container]` | 快照后得到 3 个 presentation ref，依次为 新知 / 笃行 / 领悟 |
| 关卡三张卡片图 | `img[class*=scroll_content_img]` | 按容器取不到坐标时，直接点这 3 张图 |
| 总览徽章 | `div[class*=medal_box]`、`div[class*=medal_list___]` | DOM 有虚拟化，数量会变化，不要依赖索引点徽章 |
| 徽章点亮状态 | `div[class*=medal_img]` 的 `style.backgroundImage` | 含 `medal_light` 为已点亮，灰色态不含 |
| 「+ 添加」按钮 | `div[class*=add_box]`，内层 `span[class*=add_text]` | 02/03 记录入口；07 用 /evaluate 页的「探索技能提升途径」 |
| 兴趣词条 | `div[class*=barrir_label]`、选中态 `barrir_label_action` | 04 兴趣测评 |
| 上传框 | 页面里的 `input[type=file]`（隐藏） | 先临时改为可见，再 `snapshot({selector:'input[type=file]'})` 取 ref 上传，参数名 `filePath`（单数） |

---

## 三、通用操作规范（核心提示词）

```text
【0｜先读账号「所属专业」】
涉及专业背景的内容（02 专业名称、03 领悟职业、05 职业1/2 等）不要写死：
进入任务后先读用户信息中的「所属专业」——顶部个人中心 / /mine/info，
或关卡页 div[class*=info] 中的「所属学校 / 所属专业」两个 span；
后续所有需要对口专业/职业的文本都基于它动态生成。

【1｜先快照再操作】
每次操作前先 browser_snapshot，且尽量用区域选择器：
  browser_snapshot({selector:'div[class*=scroll_content_buttons_container]'})
拿到 3 个 presentation ref 后依次点击。导航后旧 ref 全部失效，必须重新快照。

【2｜点击三级策略：标准 ref → 坐标派发 → JS 兜底】
 2a. 能拿到 ref 就用 browser_click；目标在视口外先 browser_scroll({ref, scrollIntoView:true})。
     实测普通 button / link 用 browser_click 即可。
 2b. add_text 区域与卡片图必须用「坐标派发事件」：
     「+ 添加」「探索技能提升途径」是 div + span[class*=add_text]，
     关卡卡片是 img[class*=scroll_content_img]，08 的「点击生成」是画在 img 上的文字，
     三者对 browser_click 和普通 el.click() 都可能完全无反应。
     唯一稳定流程：scrollIntoView → elementFromPoint → 派发三个鼠标事件：

  (function(){
    var el = document.querySelector('span[class*=add_text]');
    el.scrollIntoView({block:'center'});
    var rc = el.getBoundingClientRect();
    var x = rc.left + rc.width/2, y = rc.top + rc.height/2;
    var hit = document.elementFromPoint(x, y);
    if (!hit) return 'no hit';
    ['mousedown','mouseup','click'].forEach(function(t){
      hit.dispatchEvent(new MouseEvent(t,{bubbles:true,cancelable:true,view:window,clientX:x,clientY:y}));
    });
    return hit.tagName;
  })()

     要点：必须先 scrollIntoView 再取 rect，否则 elementFromPoint 返回 null；
     同名 span 常有两个（一个 0×0 隐藏），必须选 getBoundingClientRect().width > 0 的；
     1466×830 视口下卡片圆心约为 (311,711) / (729,711) / (1147,711)，08 生成图约 (710,721)；
     坐标只在同分辨率下有效，视口不同必须用 elementFromPoint 动态重算，不要照抄。
 2c. 普通 div 文字按钮用 JS 兜底（按可见文字精确匹配后 .click()，用 for 循环）：

  (function(){
    var t = '提交', els = document.querySelectorAll('div,span,p,button,a'), done = 'not found';
    for (var i = 0; i < els.length; i++) {
      if (els[i].innerText && els[i].innerText.trim() === t) {
        els[i].scrollIntoView({block:'center'});
        els[i].click();
        done = 'clicked';
        break;
      }
    }
    return done;
  })()

【3｜表单填写：browser_fill 优先，setter 兜底】
实测 input / textarea 直接用 browser_fill({ref, value}) 即可被 React 受控组件识别。
仅当 fill 无效时，才用原生 setter + 派发 input/change：

  (function(){
    var el = document.querySelector('textarea, input[type=text]');
    var proto = el.tagName === 'TEXTAREA'
      ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
    Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, '填写内容');
    el.dispatchEvent(new Event('input', {bubbles:true}));
    el.dispatchEvent(new Event('change', {bubbles:true}));
    return el.value.length;
  })()

【4｜自定义下拉选择】
Ant Design 的 select 不是原生 <select>：先 browser_click 展开按钮（通常显示「请选择」），
再用 JS 在 .ant-select-item-option 中按文字匹配并派发事件（实测最稳）：

  (function(){
    var opts = document.querySelectorAll('.ant-select-item-option');
    for (var i = 0; i < opts.length; i++) {
      if (opts[i].innerText.trim() === '本科1年级') {
        ['mousedown','mouseup','click'].forEach(function(t){
          opts[i].dispatchEvent(new MouseEvent(t,{bubbles:true,cancelable:true,view:window}));
        });
        return 'ok';
      }
    }
    return 'nf';
  })()

备选：
  ① 点选项文字对应的 button ref —— 常报
     "hit test failed: would hit div ref=eXX instead of ref=eYY"，不可靠；
  ② 键盘 ArrowDown + Enter —— ArrowDown 会先落到第 2 项，选第 1 项时不要用。
选完按 Escape 收起下拉，再点提交；否则下拉遮罩可能拦截提交按钮（表现为点击无反应）。

【5｜文件上传】
优先在本地生成真实非空 PDF（见第四节），然后按四步走：
  ① 用 browser_evaluate 把隐藏的 input[type=file] 临时改成可见：
     display='block'; opacity='1'; width='300px'; height='40px';
  ② browser_snapshot({selector:'input[type=file]'}) 取到 textbox ref；
  ③ browser_upload_file({ref, filePath:'绝对路径'})——参数名是 filePath 单个字符串，
     写成 paths 数组会报 err:undefined；
  ④ 成功判定：上传区出现文件名（如「生涯发展报告.pdf」）或自动跳下一步。
实测：直接给「点击或拖拽文件至此上传」按钮的 ref 会报
"could not accept the file directly"；只改可见性、不给宽高也可能取不到 ref。
兜底：DataTransfer + new File([Uint8Array],'name.pdf',{type:'application/pdf'})
赋给 input.files 再派发 change，也能出现文件名；但 base64 一旦截断文件就会损坏，
仅在第二个上传槽取不到 ref 时使用，且必须用完整 base64。

【6｜动作后等待再验证】
导航/提交后等 1.5~3 秒再快照或读文本；提交成功的判定文字为「提交成功」。
点击报 "Element not found" 但页面已跳转时，先读当前 URL 再决定是否重试。

【7｜表单字段自检脚本（填完后必跑）】
一次输出每个输入框的类型、占位符和已填字数，快速判断漏填/超限/填错框：

  (function(){
    var out = [], els = document.querySelectorAll('input,textarea');
    for (var i = 0; i < els.length; i++) {
      var el = els[i];
      if (el.type === 'file' || el.type === 'hidden') continue;
      out.push(el.tagName + '|' + (el.placeholder || '') + '|len=' + String(el.value || '').length);
    }
    return out.join(' @ ');
  })()

对照占位符即可定位每个框：如 07 记录页依次是 技能名称(2~20)、典型事件(20~100)、
具体行为(50~200)、预期成果(50~200)、资源(20~200)，ant-select 对应的 input 长度恒为 0 属正常。

【8｜browser_evaluate 脚本约束（实测踩坑）】
- 脚本要显式 return 原始值（字符串/数字/布尔），不要返回数组或对象
  （会序列化失败或得到 undefined，报 "An object could not be cloned"）；
- 不要写正则 /\n/、模板字符串里的转义换行，容易 SyntaxError；
  需要换行用 split(String.fromCharCode(10))；
- 少用箭头函数/展开运算符，多个值用 for 循环累加字符串拼接；
- 在 Exec 中调用时用 text(结果) 输出，环境没有 console，脚本顶层 return 会 SyntaxError；
- 工具调用失败会 reject，务必 try/catch 后 text(e.message)；
- 不要把整页 innerText、完整快照或无过滤的网络日志直接 text 出来，只取当前步骤所需字段。

【9｜失败处理与跳关】
同一动作最多重试 3 次；仍失败、页面卡住（白屏/一直 loading/点击 30 秒无响应）
或出现网络错误（系统繁忙 / ERR_FAILED），记录关卡与原因后立即进入下一关卡，
禁止原地死循环，最后统一回补。
```

---

## 四、文件类任务：生成并上传 PDF

### 4.1 生成真实中文 PDF（推荐）

用 Python 手写 Type0 / STSong-Light / UniGB-UCS2-H PDF 即可（无需第三方库），一次生成多份：
`招聘简章.pdf`、`生涯人物访谈报告.pdf`、`生涯发展报告.pdf`，每份 2~3KB，浏览器可正常上传。
生成脚本示例位置：`_tmp_assets/mkpdf.py`；核心是中文按 UTF-16BE 转十六进制，包进 `<...> Tj`，
并手写 xref / trailer。

```python
def hexs(s):
    return ''.join('%04X' % ord(c) for c in s)

cs = 'BT /F1 15 Tf 60 790 Td <' + hexs(title) + '> Tj ET '
y = 750
for line in lines:
    cs += 'BT /F1 10 Tf 60 %d Td <%s> Tj ET ' % (y, hexs(line))
    y -= 22
# 再组装 catalog / pages / page / content / font(CIDFontType0) / descriptor 与 xref
```

### 4.2 纯 JS 生成 PDF（无本地文件时的兜底）

```js
function mkPDF(title, lines){
  const hex = s => { let h=''; for (let i=0;i<s.length;i++){ h += s.charCodeAt(i).toString(16).toUpperCase().padStart(4,'0'); } return h; };
  let cs = 'BT /F1 15 Tf 60 790 Td <' + hex(title) + '> Tj ET ';
  let y = 750;
  for (const line of lines) { cs += 'BT /F1 10 Tf 60 ' + y + ' Td <' + hex(line) + '> Tj ET '; y -= 22; }
  let pdf = '%PDF-1.4 ';
  const offs = [];
  const add = (n, b) => { offs[n] = pdf.length; pdf += n + ' 0 obj ' + b + ' endobj '; };
  add(1, '<< /Type /Catalog /Pages 2 0 R >>');
  add(2, '<< /Type /Pages /Kids [3 0 R] /Count 1 >>');
  add(3, '<< /Type /Page /Parent 2 0 R /MediaBox [0 0 595 842] /Resources << /Font << /F1 5 0 R >> >> /Contents 4 0 R >>');
  add(4, '<< /Length ' + cs.length + ' >> stream ' + cs + ' endstream');
  add(5, '<< /Type /Font /Subtype /Type0 /BaseFont /STSong-Light /Encoding /UniGB-UCS2-H /DescendantFonts [6 0 R] >>');
  add(6, '<< /Type /Font /Subtype /CIDFontType0 /BaseFont /STSong-Light /CIDSystemInfo << /Registry (Adobe) /Ordering (GB1) /Supplement 2 >> /FontDescriptor 7 0 R /DW 1000 >>');
  add(7, '<< /Type /FontDescriptor /FontName /STSong-Light /Flags 6 /FontBBox [0 -200 1000 900] /ItalicAngle 0 /Ascent 880 /Descent -120 /CapHeight 880 /StemV 80 >>');
  const xp = pdf.length;
  pdf += 'xref 0 8 0000000000 65535 f ';
  for (let i = 1; i <= 7; i++) { pdf += String(offs[i]).padStart(10,'0') + ' 00000 n '; }
  pdf += 'trailer << /Size 8 /Root 1 0 R >> startxref ' + xp + ' %%EOF';
  const b = new Uint8Array(pdf.length);
  for (let i = 0; i < pdf.length; i++) { b[i] = pdf.charCodeAt(i) & 0xff; }
  return b;
}
```

### 4.3 上传要点

- 页面有多个上传框时，按 `input[type=file]` 出现顺序逐个上传，每次上传后确认文件名出现；
- 先把隐藏的 `input[type=file]` 临时改成可见（display/opacity/宽高都要设）再取 ref，最稳；
- 直接传「点击或拖拽文件至此上传」div 的 ref 会报 "could not accept the file directly"；
- 第二个上传槽取不到 ref 时，可用 DataTransfer 注入完整 base64 的 PDF 并派发 change，
  但内容截断会导致文件损坏，能用真实路径上传就不要用注入。

---

## 五、逐关 SOP（含实测字段与字数限制）

### 5.0 逐关速查卡（路由 + 字段 + 坑点）

**01 生涯愿景** `/checkpoint/first`

| 环节 | 要点 |
| --- | --- |
| 新知 | 仅浏览，无提交 |
| 笃行 | /professionalmodeledit 填 2 位榜样 → /professionalmodelshow 选榜样 → 3×textarea 各 100~300 字 |
| 领悟 | 1×textarea 100~500 字 |
| 高频坑 | 直拼 /professionalmodeledit 空白，须从关卡页点卡片；被拦先跑字段自检 |

**02 专业探索** `/checkpoint/second`

| 环节 | 要点 |
| --- | --- |
| 新知 | 仅浏览 |
| 笃行 | add_box → /record 5 字段（专业名称 ≈8，其余 100~300）→ 卡片页再点页面级「提交」 |
| 领悟 | 2×textarea 各 200~500 字（177/195 字都会被拦，写 220+） |
| 高频坑 | add_box 须坐标派发；别漏卡片页底部的页面级提交 |

**03 职业瞭望** `/checkpoint/third`

| 环节 | 要点 |
| --- | --- |
| 新知 | 仅浏览 |
| 笃行 | add_box → /record?index=1 4 字段（职业去向合计 ≤100，体会 100~500） |
| 领悟 | 2×「行业—岗位」各 ≤30 字，按「所属专业」动态生成（通用规范第 0 条） |
| 高频坑 | 同 02 的 add_box；去向超限先精简 |

**04 兴趣揭秘** `/checkpoint/fifth`（不是 fourth）

| 环节 | 要点 |
| --- | --- |
| 新知 | 仅浏览 |
| 笃行 | ① /evaluate：6 类型单选排序（label[class*=barrier_radius]）+ 词条 9 条（选中态 barrir_label_action）；② /analyse 2×50~100 字 |
| 领悟 | 2×textarea 各 200~500 字 |
| 高频坑 | /fourth 价值大罗盘是额外关卡，「系统繁忙」直接跳过 |

**05 目标锚定** `/checkpoint/sixth`

| 环节 | 要点 |
| --- | --- |
| 新知 | 必填职业1/职业2，各 2~50 字，按「所属专业」生成 |
| 笃行 | /first 平台信息+招聘简章.pdf+阅读体会 → /second 2×访谈报告.pdf → /third 实习方式 ≤20 字+2 PDF → /fourth 18 input 必填 |
| 领悟 | 1×textarea 100~500 字 |
| 高频坑 | 上传走隐藏 input 四步法；第二槽取不到 ref 用 DataTransfer 兜底；实习方式 35 字会报错 |

**06 能力盘点** `/checkpoint/seventh`

| 环节 | 要点 |
| --- | --- |
| 新知 | 必须走 /newknowledge/new1 → /new2（直访 /newknowledge 404） |
| 笃行 | /skill 3 处「去探索」→ 每处 /skillform 3×textarea 各 10~100 字 → 页面提交；成就故事 /storyfrom 7 字段必填 |
| 领悟 | 1×textarea 100~500 字 |
| 高频坑 | 弹「请先完成本页面所有技能探索」= 3 处「去探索」有漏做 |

**07 技能提升** `/checkpoint/eighth`

| 环节 | 要点 |
| --- | --- |
| 新知 | 仅浏览 |
| 笃行 | /evaluate →「探索技能提升途径」→ /evaluate/record 建 3 条（名称 2~20 / 事件 20~50 / 行为 50~200 / 成果 ≥50 / 资源 20~200 + 2 个 ant-select）→ 回列表页底部提交，见 check-circle |
| 领悟 | 1×textarea 100~500 字，残留文本整体覆盖 |
| 高频坑 | add_text 坐标派发（选宽度 >0 的）；典型事件先点「手动输入新事件」；选完下拉按 Escape |

**08 生涯启航** `/checkpoint/ninth`

| 环节 | 要点 |
| --- | --- |
| 新知 | 仅浏览长图；底部黄色圆点不可点，直接返回 |
| 笃行 | /practiceearnestly/entrance（不带 /entrance 会 404）→ 坐标点 img「点击生成」→ /growth 7 框自动汇总直接提交 |
| 领悟 | 无文本框，仅上传《生涯发展报告.pdf》；提交后不跳转，看正文「恭喜完成职业生涯闯关！」 |
| 高频坑 | 「点击生成」是图上文字，按文字匹配找不到，须坐标派发 |

### 5.1 字数限制速查表（填表前先对照）

| 关卡 | 字段 | 限制（实测） |
| --- | --- | --- |
| 01 | 榜样事迹 textarea ×3 | 各 100~300 字 |
| 01 | 领悟 ×1 | 100~500 字 |
| 02 | 记录 5 字段 | 专业名称 ≈8 字；核心课程 / 三种方法 各 100~300 字 |
| 02 | 领悟 ×2 | 各 200~500 字（实测 177/195 字都会被拦） |
| 03 | 职业去向 ×3 | 合计 ≤100 字 |
| 03 | 体会 | 100~500 字 |
| 03 | 领悟 ×2 | 各 ≤30 字，格式「行业—岗位」，按「所属专业」动态生成 |
| 04 | /analyse ×2 | 各 50~100 字 |
| 04 | 领悟 ×2 | 各 200~500 字 |
| 05 | 新知 职业1 / 职业2 | 各 2~50 字，按「所属专业」动态生成 |
| 05 | 实习实践方式 | 2~20 字（实测 35 字报错「最多填写20个字」） |
| 05 | /fourth 职业信息表 | 2 张共 18 个 input，全部必填 |
| 05 | 领悟 ×1 | 100~500 字 |
| 06 | /skillform ×3 处 | 每处 3 个 textarea，各 10~100 字 |
| 06 | /storyfrom | 7 个字段全部必填 |
| 06 | 领悟 ×1 | 100~500 字 |
| 07 | 技能名称 / 典型事件 / 具体行为 / 预期成果 / 资源 | 2~20 / 20~50 / 50~200 / ≥50 / 20~200 字 |
| 07 | 领悟 ×1 | 100~500 字（框内残留文本须整体覆盖） |
| 08 | /growth ×7 | 自动汇总 01~07 领悟，无需修改直接提交 |
| 08 | 领悟 | 无文本框，仅上传《生涯发展报告.pdf》 |

### 5.2 逐关操作步骤

```text
【01 生涯愿景】/checkpoint/first
  新知 → 笃行：/professionalmodeledit 填 2 位榜样（姓名/职业/单位）→ 提交
        → /professionalmodelshow 选择榜样 → /practiceearnestly/<id>
        → 3 个 textarea，各 100~300 字 → 提交
  → 领悟：1 个 textarea，100~500 字 → 提交

【02 专业探索】/checkpoint/second
  新知 → 笃行：/practiceearnestly → add_box → /record 5 个字段
        （限制：专业名称 ≈8 字、其余各 100~300 字；
          参考字数：核心课程 ≈66、资料查阅法 ≈139、访谈法 ≈129、实践法 ≈126）
        → 提交 → 卡片页再点「提交」
  → 领悟：2 个框，各 200~500 字（实测 177/195 字都会被拦）→ 提交

【03 职业瞭望】/checkpoint/third
  新知 → 笃行：add_box → /record?index=1 4 个字段
        （辅导员老师、专业名称、3 条职业去向 约88字、体会 约156字）→ 提交 → 卡片页再点「提交」
  → 领悟：2 个职业，每个 ≤30 字，格式「行业—岗位」→ 提交
        不要写死职业：按通用规范第 0 条读取「所属专业」，自动生成 2 条对口职业；
        例：建筑装饰工程技术 →
        「建筑装饰设计公司—室内设计师」「建筑施工企业—装饰工程项目经理」

【04 兴趣揭秘】/checkpoint/fifth（不是 fourth）
  新知 → 笃行入口有 2 张卡：
    ① 职业兴趣探索 → /evaluate：6 个兴趣类型，先点类型标签单选（实际型/艺术型/社会型等），
       再选具体词条（选中态 class 含 barrir_label_action，实测选 9 条）→ 提交成功
    ② 探索 → /analyse：2 个 textarea，各 50~100 字 → 提交成功
  → 领悟：2 个框，各 200~500 字 → 提交

  ※ /checkpoint/fourth「价值大罗盘」是额外关卡，依赖接口 work-values-list，
    接口异常时显示「系统繁忙」，与 8 枚徽章无关，直接跳过，不要在此耗费时间。

【05 目标锚定】/checkpoint/sixth
  新知：必须先填 职业1、职业2（各 2~50 字；按通用规范第 0 条读「所属专业」生成对口岗位）→ 提交成功
  笃行：/entrance → 下一步
    /first：平台/岗位信息来源（约103字）+ 上传《招聘简章.pdf》+ 阅读体会（书名+体会）→ 保存并下一步
    /second：2 个上传槽，均传《生涯人物访谈报告.pdf》→ 保存并下一步
    /third：实习实践方式 2~20 字（实测 35 字报错：「最多填写20个字」），
            再上传《生涯发展报告.pdf》《生涯人物访谈报告.pdf》→ 保存并下一步
    /fourth：2 张职业信息表共 18 个 input，全部必填 → 提交
  → 领悟：1 个 textarea，100~500 字 → 提交

【06 能力盘点】/checkpoint/seventh
  新知：/newknowledge/new1 → /new2（直接访问 /newknowledge 会 404）
  笃行：入口 → 技能探索 → /skill 有 3 处「去探索」（专业 / 职业 / 全部）：
        每处 → /skillform 3 个 textarea（专业知识 / 可迁移 / 自我管理），各 10~100 字 → 提交成功
        → 返回 /skill 点页面「提交」
      入口 → 成就故事 → /story → /storyfrom 7 个字段
        （情境S / 任务T / 行动A / 结果R + 三类技能）全部必填 → 提交 → 页面「提交」
  → 领悟：1 个 textarea，100~500 字 → 提交

【07 技能提升】/checkpoint/eighth
  新知 → 点笃行卡片直接进入 /evaluate（记录列表页）
    → 点「探索技能提升途径」（新增记录入口）：browser_click 无效，
      必须用「坐标派发事件」点内层 span[class*=add_text]（见通用规范第 2 条）→ /evaluate/record
    → 第 1 条填完点「提交」保存，自动返回 /evaluate 列表并显示卡片（编辑 / 删除）
    → 重复「探索技能提升途径」建第 2、3 条
    字段顺序与限制（按页面占位符）：
      技能类型（ant-select：专业知识技能 / 可迁移技能 / 自我管理技能）
      技能名称 2~20 字（input）
      典型事件：先点「手动输入新事件」，才会出现 20~100 字输入框；
                实测填 99 字被截断为 50 字，建议按 20~50 字写
      具体行为 50~200 字（textarea）；预期成果 ≥50 字（textarea，建议 70 字以上）
      预计时间（ant-select：高职1~3年级 / 本科1~5年级）
      资源 20~200 字（textarea）
    填完后跑「字段自检脚本」核对 6 个框都有值且字数达标，再点提交
    实测建 3 条：专业软件应用能力（专业知识）/ 沟通表达能力（可迁移）/ 时间管理能力（自我管理）
    → 回 /evaluate 列表页点页面底部「提交」（按钮可能在视口外，browser_click 可直接点）
      → 跳回 /checkpoint/eighth，出现 check-circle 即成功
  → 领悟：/checkpoint/eighth/comprehend，1 个 textarea 100~500 字
    （进入时框内可能有自动拼接的残留文本；browser_fill 直接整体覆盖，实测 350 字通过）
    → 点「提交」→ 回 /mine/recruit（不会停留在 comprehend 页）

【08 生涯启航】/checkpoint/ninth
  新知：/checkpoint/ninth/newknowledge（仅浏览长图，无提交按钮）
        ※ 底部蓝色横幅上的黄色圆点（img[class*=scroll_below]）点了没反应，
          不要在这里反复点击；直接返回 /checkpoint/ninth 再点笃行卡片。
  笃行：/checkpoint/ninth/practiceearnestly/entrance
        → 「点击生成」是画在 img 上的文字，按文字匹配 JS 找不到；
          用坐标派发事件点这张图（1466×830 视口下圆心约 (710,721)），
          或 browser_click 该 img 的 presentation ref → 跳 /growth
        → /checkpoint/ninth/practiceearnestly/growth：
          7 个 textarea 自动汇总 01~07 的领悟内容
          （实测字数 290/595/89/625/400/346/404），无需修改，直接点「提交」
        → 提交后回 /entrance，出现 check-circle 即成功
  领悟：/checkpoint/ninth/comprehend
        → 本页只有上传框、没有 textarea；上传《生涯发展报告.pdf》
          （按通用规范第 5 条：先把隐藏 input 设为可见 → snapshot 取 ref → upload_file）
        → 点「提交」后页面不跳转，正文出现
          「恭喜完成职业生涯闯关！预祝你在未来的职业道路上一帆风顺」即成功
```

每关通用节奏：

1. `browser_navigate` 进入关卡引导页；
2. 点「新知」卡片读完（部分关卡需进入子页才解锁后续）；
3. 点「笃行」卡片，按子步骤填表 / 上传 / 生成；
4. 点「领悟」卡片，写总结或上传报告，点「提交」；
5. 提交成功后再进入下一关，最后统一回 `/mine/recruit` 核对徽章。

### 5.3 逐关失败排查决策树

按「症状 → 判定 → 处理」逐级走，每条链路最多重试 3 次，仍失败记入待回补清单并跳关。

```text
【01 生涯愿景】
直接访问 /professionalmodeledit 空白？
  → 不要拼 URL，回 /checkpoint/first 点「笃行」卡片进入
笃行提交被拦？
  → 跑字段自检脚本：3 个 textarea 是否都在 100~300 字
  → 榜样未保存？回 /professionalmodelshow 确认 2 位榜样已选并已提交
领悟提示"请先完成新知和笃行"？
  → 补读新知页再回来提交
点提交无反应？
  → 重新 snapshot 取新 ref → scrollIntoView → 再点；≤3 次后 JS 文字兜底

【02 专业探索】
add_box「+ 添加」点不动？
  → 坐标派发：scrollIntoView → elementFromPoint → mousedown/mouseup/click
  → 命中元素宽度必须为 >0（同名 span 常有 0×0 隐藏）
/record 保存被拦？
  → 对照 5.1 速查表：专业名称 ≈8 字、其余 100~300 字
提交后回不到卡片页 / 卡片页找不到「提交」？
  → 回 /practiceearnestly 列表页底部找页面级提交按钮
领悟被拦？
  → 2 个框各 200~500 字（177/195 字都不够，先扩到 220+）

【03 职业瞭望】
add_box 点不动？
  → 同 02 坐标派发
「职业去向」被拦？
  → 3 条合计 ≤100 字，超了就精简
领悟被拦？
  → 每条 ≤30 字且格式必须是「行业—岗位」
  → 内容不写死：先读「所属专业」再生成对口职业

【04 兴趣揭秘】
兴趣类型点不动 / 点了没反应？
  → 它是 label[class*=barrier_radius] 不是 button，用 JS 派发鼠标事件
词条选了不算数？
  → 确认词条 class 已变成 barrir_label_action；没变说明没点上，重选
/analyse 提交被拦？
  → 2 个框各 50~100 字
/fourth 价值大罗盘显示"系统繁忙"？
  → 额外关卡、接口常故障，直接跳过，不影响徽章

【05 目标锚定】
新知无法提交？
  → 职业1、职业2 是否都已填（各 2~50 字）；不填职业后续 /growth 汇总会缺料
上传按钮报 "could not accept the file directly"？
  → 隐藏 input 四步法：设可见（含宽高）→ snapshot 取 ref → upload_file
第二个上传槽取不到 ref？
  → DataTransfer + new File 兜底（必须用完整 base64，截断即损坏）
「实习实践方式」报错？
  → 2~20 字（实测 35 字报错），精简后重交
/fourth 提交被拦？
  → 字段自检脚本核对 18 个 input 全部有值

【06 能力盘点】
访问 /newknowledge 404？
  → 必须依次走 /newknowledge/new1 → /newknowledge/new2
/skill 点提交弹「请先完成本页面所有技能探索」？
  → 3 处「去探索」（专业/职业/全部）全部完成；漏哪处补哪处
/skillform 被拦？
  → 3 个 textarea 各 10~100 字
/storyfrom 被拦？
  → 7 个字段全部必填（情境/任务/行动/结果 + 三类技能）

【07 技能提升】
「探索技能提升途径」点了没反应？
  → 坐标派发，且确认选中宽度 >0 的 span[class*=add_text]
典型事件的输入框不出现？
  → 先点「手动输入新事件」，框才会渲染出来
「典型事件」字数被拦？
  → 按 20~50 字写（实测 99 字被截断为 50）
「预期成果」被拦？
  → ≥50 字，建议直接写 70+ 字
下拉选不中 / 提交被下拉遮罩拦截？
  → .ant-select-item-option 文字匹配派发事件 → 按 Escape 收下拉 → 再提交
填完提交被拦？
  → 字段自检脚本核对 6 个框（select 对应 input 恒为 0 属正常）
3 条建完后徽章没变？
  → 回 /evaluate 列表页点底部页面级「提交」，跳回 /checkpoint/eighth 出现 check-circle 才算完
领悟框里已有文字？
  → 是残留自动文本，browser_fill 整体覆盖，不要追加

【08 生涯启航】
/practiceearnestly 404？
  → 必须带 /entrance 后缀
「点击生成」按文字找不到？
  → 它是画在 img 上的文字；坐标派发点 img 中央（1466×830 视口约 710,721）
  → 或 browser_click 该 img 的 presentation ref
/growth 页 7 个框要改吗？
  → 自动汇总 01~07 领悟，直接提交；若某框为空，说明对应关卡领悟没交，回补后再来
新知页底部黄色圆点点了没反应？
  → 正常，它不是按钮；返回 /checkpoint/ninth 点笃行卡片
领悟提交后页面不跳转？
  → 正常；读正文出现「恭喜完成职业生涯闯关！」即成功，再回 /mine/recruit 验徽章
```

---

## 六、异常处理（提示词中的容错条款）

```text
—— 回补清单（统一记录格式）——
卡关时按此格式记录：关号 / 环节（新知·笃行·领悟）/ 症状 / 已重试次数 / 待回补动作
例：07 / 笃行 / 「预期成果」48 字被拦 / 3 次 / 扩到 70 字后重交
全部关卡走完后按清单逐条回补，回补一项核销一项。

—— 环境与导航 ——
1) 报 "No browser page is open" 或 browser_tabs 为空
   → 标签页被关闭，重新 browser_navigate 打开目标 URL 继续。

2) 访问某路径 404
   → 不要拼 URL，从 /mine/recruit 或上级关卡页逐级点击进入，记录真实路径。
   → 已知：06 新知走 /newknowledge/new1、/new2；08 笃行走 /practiceearnestly/entrance。

3) 误跳转（点 A 跳到 B 页）
   → 说明 ref 指向错误元素，改用 JS 按文字定位点击，并重新确认当前 URL。
   → 实测：点总览徽章容易误跳 /mine/info，徽章不稳定时直接按已核实 URL 导航。

4) 某关卡住（白屏、一直 loading、点击超过 30 秒无响应）
   → 用 browser_navigate 重新打开当前 URL 重试 1 次；仍无响应则记录该关及卡点，
     直接 browser_navigate 进入下一关卡，继续后面的流程，不得在本关卡死循环。

5) 页面被"检测到您当前打开的页面不适配手机端使用"全屏遮挡
   → ref 点击无效，用 JS 点内部按钮：
     var e=document.querySelector('span[class*=mobile_btn]'); if(e){e.click();}
     关闭后再继续操作。

—— 表单与提交 ——
6) 领悟页提示"请先完成 新知 和 笃行 环节"
   → 先补做该关新知（含 /new1、/new2 子页），再回来提交。

7) 点击提交无反应
   → 重新 snapshot 取新 ref → 必要时 scrollIntoView → 再点；
   → 仍无效则改用 innerText 精确匹配的 JS 点击；最多重试 3 次。

8) 提交报字数不足 / 超限
   → 读取提示中的字符数要求，对相应字段扩容或精简后重新提交。
   → 常见：05 实习实践方式 ≤20 字；07 典型事件 ≥20 字、预期成果 ≥50 字。

9) 接口持续 ERR_FAILED / 页面"系统繁忙" / 网络错误
   → 判定为服务端或网络故障，不要死循环重试；记录关号与原因，立即跳到下一关卡，最后回补。

—— 脚本与上传 ——
10) browser_evaluate 报 "An object could not be cloned" / "Script failed to execute"
   → 脚本只返回原始值；去掉正则、箭头函数、展开运算符与对象/数组返回值，
     改用 for 循环拼接字符串。

11) 页面出现多个上传框
   → 按 input[type=file] 顺序逐个上传，每次上传后等待并确认文件名出现。

—— 控件特性 ——
12) browser_click 选项报 "hit test failed: would hit div ref=eXX instead of ref=eYY"
   → 下拉选项互相遮挡，改用 .ant-select-item-option 文字匹配 + 派发 mousedown/mouseup/click。

13) 「+ 添加」「探索技能提升途径」点击无反应
   → 用坐标派发事件（scrollIntoView → elementFromPoint → mousedown/mouseup/click），
     并确认选中的是宽度 >0 的那个 span[class*=add_text]。

—— 验收注意 ——
14) 08 领悟提交后仍停在 /checkpoint/ninth/comprehend
   → 正常现象，页面不会自动跳转；读正文是否出现「恭喜完成职业生涯闯关！」判定成功，
     再回 /mine/recruit 核验徽章。

15) 完成 8 关后没看到「恭喜完成职业生涯闯关！」弹窗
   → 弹窗只在首次完成瞬间出现，重载不再弹；以徽章点亮状态为验收依据，不要反复刷新。
```

---

## 七、验收

```text
最后访问 https://zgs.chsi.com.cn/mine/recruit：

1. 目视：8 个关卡徽章全部为彩色（未通关为灰色）；
2. 脚本核验（推荐，实测返回 "11111111|n=8"）：
   (function(){
     var els = document.querySelectorAll('div[class*=medal_img]'), out = '';
     for (var i = 0; i < els.length; i++) {
       out += (String(els[i].style.backgroundImage || '').indexOf('medal_light') >= 0 ? '1' : '0');
     }
     return out + '|n=' + els.length;
   })()
   → 期望 8 个 "1"；元素数量不足说明页面还没渲染完，等 2 秒重取一次再判断
3. 页面不再出现任何「去闯关」文字
   （读正文即可：document.body.innerText.indexOf('去闯关') >= 0 应为 false）；
4. 抽查 08 成长单可下载（我的生涯探索成长单.docx）；
5. 「恭喜完成职业生涯闯关！」弹窗只在首次完成时出现，不作为必验项。

若仍有灰色徽章：按第二节映射定位该关缺失环节补做（注意 fourth 是额外关卡，不计入）。
```

---

## 八、一键复用提示词（可直接粘贴）

```text
请使用内置浏览器，登录状态下完成 https://zgs.chsi.com.cn/mine/recruit 的 8 个生涯闯关关卡。
规则：
1. 登录页需人工完成，调用 browser_waiting_for_user_interaction 等待用户登录后再开始。
2. 从首页点徽章进入（徽章点不稳时直接按已核实 URL 导航），逐关完成 新知 → 笃行 → 领悟 三段并提交。
   关卡映射：first=01，second=02，third=03，fifth=04，sixth=05，seventh=06，eighth=07，ninth=08；
   fourth 是「价值大罗盘」额外关卡，不属于 8 枚徽章，接口常故障，直接跳过。
3. 每次操作前先 browser_snapshot；点击分三级：先 browser_click ref；
   「+ 添加」「探索技能提升途径」等 add_text 区域、卡片图、08 的「点击生成」图点不动时，
   统一用「scrollIntoView → elementFromPoint → 派发 mousedown/mouseup/click」的坐标点击法
   （按文字找元素时选宽度 >0 的那个，同名 span 常有一个 0×0 隐藏）；
   普通 div 文字按钮再用 innerText 精确匹配 .click() 兜底。
4. 表单赋值优先用 browser_fill（实测受控输入可直接生效）；无效时再用原生 value setter
   + 派发 input/change。ant-select 先点开，再用 JS 在 .ant-select-item-option 中按文字匹配
   并派发 mousedown/mouseup/click（点 option/button ref 常报 hit test failed）；
   选完按 Escape 收起下拉再提交，避免遮罩拦截；填完跑字段自检脚本核对字数。
5. 需要上传材料时，本地生成真实中文 PDF；先把隐藏的 input[type=file] 临时改成可见，
   再 browser_snapshot({selector:'input[type=file]'}) 取 ref，
   用 browser_upload_file({ ref, filePath: '绝对路径' }) 上传（参数是 filePath 单数）；
   直接点「点击或拖拽文件至此上传」会报 could not accept the file directly。
   招聘简章、访谈报告、实习证明、生涯发展报告各准备一份并复用；成功判定是上传区出现文件名。
6. 需要按用户背景生成的内容（02 专业名称、03 领悟职业、05 职业1/2），
   先读页面「所属专业」（/mine/info 或关卡页 div[class*=info] 的 span）再动态生成。
7. 每次提交后等 1~3 秒再读页面提示验证结果；失败重新取 ref 重试，最多 3 次；
   若某关卡住（白屏、一直加载、点击无响应）或遇接口报错（系统繁忙 / ERR_FAILED）/ 网络错误，
   按回补模板记录后立即跳到下一关卡，最后统一回补。
8. 遇阻先对照 5.1 字数速查表（字段限制）与 5.3 逐关决策树（症状→处理）定位；
   仍无解再按第六节异常条款处理。
9. 被"不适配手机端"全屏遮挡时，用 JS 点 span[class*=mobile_btn] 关闭。
10. browser_evaluate 只返回原始值，避免正则、箭头函数和数组/对象返回。
11. 完成后回到 /mine/recruit，用 document.querySelectorAll('div[class*=medal_img]') +
    backgroundImage 是否含 medal_light 核验，期望 "11111111|n=8"；
    并确认正文无「去闯关」文字（完成弹窗只出现一次，不作为必验项）。
12. 全程用中文汇报每关完成情况与遗留问题。
```

---

## 九、版本记录

- v1 初版：首次跑通 8 关后的流程记录；
- v2 实测修订版：两轮跑通后校正路由、选择器、字数限制、下拉与上传；
- v3 实测修订版：补充 07/08 细节、坐标点击法、隐藏上传框取 ref 流程；
- v4 逻辑整理版：重构章节顺序（总览→结构→通用规范→PDF→SOP→异常→验收→一键提示词）、
  合并 browser_evaluate 重复约束、统一填表口径（browser_fill 优先 / setter 兜底）、
  异常处理按 环境/表单/脚本/控件/验收 分组；03 领悟示例改为按账号「所属专业」动态生成；
  全文去除个人身份信息，示例中性化。
- v5 速查增强版：第五节新增「字数限制速查表」，8 关全部字段的限制集中对照；
  SOP 正文保留实测细节，速查表负责快速定位，两者互补。
- v5.1 决策树版：新增 5.3「逐关失败排查决策树」，每关按「症状 → 判定 → 处理」
  组织实测踩过的坑，与第六节通用异常条款配合（决策树管关卡特异问题，异常条款管通用问题）。
- v5.2 一致性修订版：02 记录字段区分「限制/参考字数」；新增通用规范第 0 条（读所属专业），
  03/05 专业相关内容统一引用；坐标加注视口前提；一键提示词扩至 12 条并引用速查表/决策树；
  第六节新增回补清单统一记录格式。
- v5.3 速查卡版：第五节开头新增 5.0「逐关速查卡」，8 关各一张（路由+新知/笃行/领悟要点+高频坑），
  与 5.1 字数表、5.2 SOP、5.3 决策树构成「卡片速览 → 字数对照 → 详细步骤 → 失败排查」四层。
