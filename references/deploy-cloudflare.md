# 部署与跨设备同步（Cloudflare，免费）

## 一、纯静态版（几分钟上线，无跨设备同步）

1. 登录 dash.cloudflare.com → Workers & Pages → Create application → **Upload your static files**。
2. 把 AI 生成的网站文件命名为 `index.html` 上传，Deploy 后得到
   `https://<项目名>.<你的子域名>.workers.dev`。
3. 以后每周更新：让 AI 重新生成后，重新上传同名文件即可，链接不变。
   （数据存浏览器 localStorage，每个浏览器各存各的，重部署不清空。）

## 二、跨设备同步版（推荐，仍免费）

把纯静态页升级为 **Cloudflare Worker + Workers KV**：复习清单 / 单词 / 其他知识点 / 课堂笔记
的读写全部走云端接口，任何设备、任何浏览器打开同一链接就读写同一份数据。
免费额度（每天 10 万次读 / 1 千次写）对个人使用完全够。

### 部署目录结构

```
deploy/
├── worker.js          # API 路由（/api/data 读写 KV，其余回退静态资源）
├── wrangler.jsonc     # Worker 配置（assets + KV 绑定）
└── public/
    └── index.html     # 网页本体（就是 Skill 生成的单文件 HTML）
```

### wrangler.jsonc

```jsonc
{
  "name": "coursehub",
  "main": "worker.js",
  "compatibility_date": "2025-06-01",
  "assets": { "directory": "./public", "binding": "ASSETS" },
  "kv_namespaces": [
    { "binding": "STUDY_KV", "id": "在这里填你的 namespace id" }
  ]
}
```

先在 Cloudflare 控制台创建一个 KV namespace（Workers & Pages → Workers KV → Create），
把它的 id 填进上面。

### worker.js

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === '/api/data') {
      const KEY = 'study_data';
      if (request.method === 'GET') {
        const data = await env.STUDY_KV.get(KEY, 'json');
        return new Response(JSON.stringify(data || {}), {
          headers: { 'Content-Type': 'application/json;charset=utf-8', 'Cache-Control': 'no-store' }
        });
      }
      if (request.method === 'PUT') {
        const body = await request.json();
        await env.STUDY_KV.put(KEY, JSON.stringify({
          v: Date.now(),
          review: body.review || [],
          extra: body.extra || {},
          vocab: body.vocab || [],
          notes: body.notes || []
        }));
        return new Response('{"ok":true}', {
          headers: { 'Content-Type': 'application/json;charset=utf-8', 'Cache-Control': 'no-store' }
        });
      }
      return new Response('{"ok":false}', { status: 405 });
    }
    return env.ASSETS.fetch(request);
  }
};
```

### 网页端（index.html 末尾 <script>）加三件事

1. **pushToCloud()**：把 `{review, extra, vocab, notes}` 用
   `fetch('/api/data', {method:'PUT', ...})` 上传（800ms 防抖）。
2. **loadFromCloud()**：页面加载时 GET 云端数据，按**版本号方向判断**决定以哪边为准（见下）。
3. 每次本地保存（加词 / 加知识点 / 加笔记 / 改清单）后调用 `schedulePush()` 触发上传。

### 同步方向判断（重要，直接抄这个逻辑）

本地保存「上次确认的云端版本号」`com5104_sync_ts`：

- 云端 `v > sync_ts` → 云端较新，**以云端覆盖本地**（这样「设备 A 删掉的条目，设备 B 打开也会消失」）；
- 否则 → 本地较新或相同，**以本地为准推回云端**（刚删除 / 刚添加立即同步出去）；
- 首次升级（本地没有 sync_ts）→ 做一次「云端 ∪ 本地」合并建立基线，任何一方旧数据都不丢；
- 请求失败时提示「⚠ 离线」并继续用本地。

## 三、踩坑提醒

- **Cloudflare KV 是最终一致性存储**：写入后立刻读可能拿到旧值（数十秒内全球收敛）。
  所以不要用「云端覆盖本地」或「云端 ∪ 本地 合并」两种简单做法：
  前者会让刚添加的数据刷新后「闪没」；后者会让删除不传播（A 删了，B 的本地缓存合并时又把旧条目复活）。
  用上面的「版本号方向判断」。
- Dashboard 网页上给「Upload assets」类型的 Worker 添加 KV 绑定经常失败（点击无反应），
  直接用 **wrangler CLI** 最稳：

  ```bash
  npx wrangler login      # 首次授权一次（浏览器里点 Authorize）
  cd deploy
  npx wrangler deploy     # 之后每周更新：覆盖 public/index.html 后重跑这条
  ```

- 每次更新后链接不变、数据不丢（云端数据与 Worker 绑定都保持不变）。
