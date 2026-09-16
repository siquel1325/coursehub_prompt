# 部署与跨设备同步（Cloudflare，免费）

> ⚠ 动手部署前，先问使用者：「需要我帮你免费部署上线、并开启跨设备同步吗？」
> 需要 → 按下面「跨设备同步版」执行；不需要 → 告知「纯本地文件，数据只在当前浏览器
> localStorage，换设备/浏览器不同步」，不需要执行本文件。

## 一、纯静态版（几分钟上线，无跨设备同步）

1. 登录 dash.cloudflare.com → Workers & Pages → Create application → **Upload your static files**。
2. 把 AI 生成的网站文件命名为 `index.html` 上传，Deploy 后得到
   `https://<项目名>.<你的子域名>.workers.dev`。
3. 以后每周更新：让 AI 重新生成后，重新上传同名文件即可，链接不变。
   （数据存浏览器 localStorage，每个浏览器各存各的，重部署不清空。）

## 二、跨设备同步版（推荐，仍免费）

把纯静态页升级为 **Cloudflare Worker + Workers KV + R2**：复习清单 / 单词 / 其他知识点 /
课堂笔记 / 富文本编辑 / 日历事件 / 课堂图片的读写全部走云端接口，任何设备、任何浏览器打开
同一链接就读写同一份数据（删除也会同步传播）。免费额度（KV：每天 10 万次读 / 1 千次写；
R2：免费 10GB 存储 + 每月 100 万次读 A 类操作 / 1000 万次写 B 类操作）对个人使用完全够。

### 部署目录结构

```
deploy/
├── worker.js          # API 路由（/api/data 读写 KV、/api/image/* 读写 R2，其余回退静态资源）
├── wrangler.jsonc     # Worker 配置（assets + KV 绑定 + R2 bucket）
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
  ],
  "r2_buckets": [
    { "binding": "IMAGES", "bucket_name": "coursehub-images" }
  ]
}
```

先在 Cloudflare 控制台创建两个资源：
1. 一个 KV namespace（Workers & Pages → Workers KV → Create），把它的 id 填进上面；
2. 一个 R2 bucket（R2 → Create bucket，名字与上面 bucket_name 一致，如 coursehub-images）；
   也可以用 wrangler CLI 自动创建：`npx wrangler r2 bucket create coursehub-images`。

### worker.js

```js
const KV_KEY = 'course_data';

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const path = url.pathname;

    // ---- 结构化数据 API（KV）----
    if (path === '/api/data') {
      if (request.method === 'GET') {
        const data = await env.STUDY_KV.get(KV_KEY, 'json');
        return new Response(JSON.stringify(data || {}), {
          headers: { 'Content-Type': 'application/json; charset=utf-8', 'Cache-Control': 'no-store' }
        });
      }
      if (request.method === 'PUT') {
        let body;
        try { body = await request.json(); } catch (e) {
          return new Response('bad json', { status: 400 });
        }
        // 7 个同步字段，缺省给空值（防脏数据）
        const data = {
          v: Date.now(),
          review: Array.isArray(body.review) ? body.review : [],
          extra: (body.extra && typeof body.extra === 'object') ? body.extra : {},
          vocab: Array.isArray(body.vocab) ? body.vocab : [],
          notes: Array.isArray(body.notes) ? body.notes : [],
          edits: (body.edits && typeof body.edits === 'object') ? body.edits : {},
          calendar: Array.isArray(body.calendar) ? body.calendar : [],
          images: (body.images && typeof body.images === 'object') ? body.images : {}
        };
        await env.STUDY_KV.put(KV_KEY, JSON.stringify(data));
        return new Response(JSON.stringify({ ok: true }), {
          headers: { 'Content-Type': 'application/json; charset=utf-8' }
        });
      }
      return new Response('method not allowed', { status: 405 });
    }

    // ---- 图片对象 API（R2）----
    if (path.startsWith('/api/image/')) {
      const id = decodeURIComponent(path.slice('/api/image/'.length));
      if (!id) return new Response('bad id', { status: 400 });
      if (request.method === 'GET') {
        const obj = await env.IMAGES.get(id);
        if (!obj) return new Response('not found', { status: 404 });
        const headers = new Headers();
        headers.set('Content-Type', obj.httpMetadata && obj.httpMetadata.contentType
          ? obj.httpMetadata.contentType : 'image/jpeg');
        headers.set('Cache-Control', 'public, max-age=86400');
        return new Response(obj.body, { headers });
      }
      if (request.method === 'PUT') {
        const body = await request.arrayBuffer();
        if (body.byteLength > 8 * 1024 * 1024) {
          return new Response('image too large (max 8MB after compression)', { status: 413 });
        }
        await env.IMAGES.put(id, body, { httpMetadata: { contentType: 'image/jpeg' } });
        return new Response(JSON.stringify({ ok: true }), {
          headers: { 'Content-Type': 'application/json; charset=utf-8' }
        });
      }
      if (request.method === 'DELETE') {
        await env.IMAGES.delete(id);
        return new Response(JSON.stringify({ ok: true }), {
          headers: { 'Content-Type': 'application/json; charset=utf-8' }
        });
      }
      return new Response('method not allowed', { status: 405 });
    }

    // ---- 静态资源（public/ 目录）----
    return env.ASSETS.fetch(request);
  }
};
```

### 网页端（index.html 末尾 <script>）加三件事

1. **pushToCloud()**：把 `{review, extra, vocab, notes, edits, calendar, images}` 用
   `fetch('/api/data', {method:'PUT', ...})` 上传（800ms 防抖）。
2. **loadFromCloud()**：页面加载时 GET 云端数据，按**版本号方向判断**决定以哪边为准（见下）。
3. 每次本地保存（加词 / 加知识点 / 加笔记 / 改清单 / 编辑保存 / 日历增删 / 图片增删）后调用
   `schedulePush()` 触发上传。
4. **图片同步**：上传图片时先 PUT 到 `/api/image/<id>`（R2），元数据写进 images 字段随
   /api/data 上传；打开页面时遍历 images 元数据，本地 IndexedDB 没有的图片从
   `/api/image/<id>` 拉取缓存；删除时本地 + R2 一起删。
   图片压缩上限：单张 PUT 限 8MB（前端压缩后通常 200–400KB，不会触顶）。

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
- 若使用者反馈「一个浏览器删除了，另一个浏览器还有」：说明同步方向判断没生效或没部署。
  已部署时删除会随「云端覆盖本地」传播（云 v > 本地 sync_ts 即覆盖，删除的条目不会复活）；
  未部署时多浏览器各存各的，必然不同步——回到「二」完成部署。
- R2 免费额度（约 10GB 存储）对课堂照片足够；前端已压缩，一般不需要额外限制张数。
