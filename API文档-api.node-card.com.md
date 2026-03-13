# 开放API对接文档（第三方兑换站接入）

更新时间：2026-03-14  
适用对象：使用你方网站/系统对接本平台 API 的开发者

## 1. 对接范围

本对接文档仅包含以下 5 个接口：

1. `POST /api/open/card/redeem` 兑换卡
2. `GET|POST /api/open/card/status` 查询卡状态
3. `POST /api/open/card/transactions` 查询交易
4. `POST /api/open/merchant/list` 商户列表
5. `GET|POST /api/open/platform/capacity` 平台容量

## 2. 请求与返回规范

基础地址：
```text
https://api.node-card.com
```

统一返回格式：

```json
{
  "code": 1,
  "msg": "",
  "time": 1770000000,
  "data": {}
}
```

字段说明：

| 字段 | 说明 |
|---|---|
| `code` | `1=成功`，`0=失败` |
| `msg` | 提示信息 |
| `time` | 服务端时间戳（秒） |
| `data` | 业务数据 |

建议请求头：

```text
Content-Type: application/x-www-form-urlencoded
```

## 3. 接口详情

### 3.1 商户列表

`POST /api/open/merchant/list`

请求参数：无

成功返回 `data`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 商户数组 |
| `list[].id` | int | 商户ID |
| `list[].name` | string | 商户名称 |

示例：

```json
{
  "code": 1,
  "msg": "",
  "time": 1770000000,
  "data": {
    "list": [
      { "id": 1, "name": "Google" },
      { "id": 2, "name": "Netflix" }
    ]
  }
}
```

---

### 3.2 平台容量

`GET /api/open/platform/capacity`  
`POST /api/open/platform/capacity`

请求参数：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `platform_id` | int | 否 | 平台ID，不传返回全部平台 |

成功返回 `data`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | array | 平台容量数组 |
| `list[].platform_id` | int | 平台ID |
| `list[].capacity_rate` | number | 容量率百分比，`-1` 表示该平台当前不可用 |
| `list[].card_head_num` | string | 卡头号 |
| `list[].region` | string | 地区 |

---

### 3.3 查询卡状态

`GET /api/open/card/status`  
`POST /api/open/card/status`

请求参数：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `card_key` | string | 否 | 卡密；为空或不存在时返回 `none` |

成功返回 `data`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `status` | string | `none` / `unused` / `redeeming` / `used` |
| `used_time` | int | 使用时间戳（秒），无则 `0` |

示例：

```json
{
  "code": 1,
  "msg": "",
  "time": 1770000000,
  "data": {
    "status": "used",
    "used_time": 1770000000
  }
}
```

---

### 3.4 兑换卡

`POST /api/open/card/redeem`

请求参数：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `card_key` | string | 是 | 卡密 |
| `merchant_dict_id` | int | 否 | 商户ID；当模板限制商户时必填 |
| `platform_id` | int | 否 | 指定兑换平台ID；不传则按卡密模板所属平台 |

成功返回 `data`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `card_number` | string | 卡号 |
| `cvv` | string | CVV |
| `exp` | string | 有效期（如 `03/29`） |
| `expire_time` | int | 到期时间戳（秒） |
| `redeem_time` | int | 兑换时间戳（秒） |
| `available_hours` | int | 剩余小时 |
| `is_limit_merchant` | bool | 是否限制商户 |
| `limit_merchant` | string | 限制商户名 |
| `merchant_name` | string | 商户名 |
| `available_amount` | number | 金额 |
| `full_billing_address` | string | 账单地址 |

说明：


- 已使用且未过期的卡密，会返回已兑换卡信息

---

### 3.5 查询交易

`POST /api/open/card/transactions`

请求参数：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `card_key` | string | 是 | 卡密 |

成功返回 `data`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `transactions` | array | 交易记录数组 |
| `transactions[].id` | int | 序号（从1开始） |
| `transactions[].date` | string | 交易时间 |
| `transactions[].merchant` | string | 商户名 |
| `transactions[].amount` | string | 金额+币种（如 `-1 USD`） |
| `transactions[].status` | string | 交易状态 |
| `transactions[].content` | string | 交易内容 |
| `transactions[].failureReason` | string | 失败原因（无则空字符串） |

## 4. 错误处理规则

业务输入问题（直接返回可给用户看的提示）示例：

- `卡密不存在！`
- `该卡密对应卡片已到期`
- `当前模板限制商户，请先选择商户`
- `该商户未配置当前平台的商户ID`
- `该卡头暂时停用，请选择其他卡头，或者等恢复后再试`

非输入问题（内部错误）统一返回：

```text
服务器错误，请稍后再试！
```

## 5. 接入流程建议（第三方网站）

1. 页面加载时调用商户列表和平台容量接口，渲染下拉选择与容量提示。  
2. 用户输入卡密后先调用卡状态接口（可选，但推荐）。  
3. 调用兑换接口拿卡信息。  
4. 用户查看交易时调用交易接口。  
5. 前端仅根据 `code` 判定成功失败，`msg` 直接展示给用户。

## 6. cURL 示例

兑换卡：

```bash
curl -X POST 'https://your-domain.com/api/open/card/redeem' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'card_key=9e366fa0-adf3-47d2-bbc1-f2749bf9dacd&merchant_dict_id=1&platform_id=1'
```

查询卡状态：

```bash
curl 'https://your-domain.com/api/open/card/status?card_key=9e366fa0-adf3-47d2-bbc1-f2749bf9dacd'
```

查询交易：

```bash
curl -X POST 'https://your-domain.com/api/open/card/transactions' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'card_key=9e366fa0-adf3-47d2-bbc1-f2749bf9dacd'
```

