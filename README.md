
兑换前端（Vue 3 + Element Plus + Vite）。

[API对接文档](API文档-api.node-card.com.md)
## 特性

- 单个兑换、批量兑换、消费记录查询、只查询不兑换
- API 基地址默认 `https://api.node-card.com`
- 对接需要提供域名白名单，添加跨域访问白名单后才能直接调用。
- 公告内容可使用自定义的接口地址获取，或者在本项目中直接设置为静态公告文本后再打包，具体设置：在"src\App.vue"中配置state.noticeText的值
## 开发

```bash
npm install
npm run dev
```

## 打包

```bash
npm run build
```

产物目录：`dist/`

## 接口配置

默认使用：

```env
VITE_API_BASE=https://api.node-card.com
```

如需改为其他域名，新建 `.env` 或 `.env.production` 覆盖该变量即可。也可以直接在"src\config.js"中更改API_BASE值。
