# DPI (深度包检测) 工作原理

## 一、DPI 主入口

```c
// app_filter.c:1207-1214
int dpi_main(struct sk_buff *skb, flow_info_t *flow) {
    dpi_http_proto(flow);    // 解析 HTTP
    dpi_https_proto(flow);   // 解析 HTTPS (SNI)
    return 0;
}
```

---

## 二、HTTP 协议解析

### 解析流程

```c
// app_filter.c:735-791
void dpi_http_proto(flow_info_t *flow)
{
    // 1. 查找换行符 0x0d 0x0a (\r\n)
    for (i = 0; i < data_len; i++) {
        if (data[i] == 0x0d && data[i + 1] == 0x0a) {
            
            // 2. 识别 HTTP 方法
            if (0 == memcmp(&data[start], "POST ", 5)) {
                flow->http.match = AF_TRUE;
                flow->http.method = HTTP_METHOD_POST;
                flow->http.url_pos = data + start + 5;
                flow->http.url_len = i - start - 5;
            }
            else if (0 == memcmp(&data[start], "GET ", 4)) {
                flow->http.match = AF_TRUE;
                flow->http.method = HTTP_METHOD_GET;
                flow->http.url_pos = data + start + 4;
                flow->http.url_len = i - start - 4;
            }
            
            // 3. 提取 Host 头
            else if (0 == memcmp(&data[start], "Host:", 5)) {
                flow->http.host_pos = data + start + 6;
                flow->http.host_len = i - start - 6;
            }
            
            // 4. 检测双重换行 (头部结束)
            if (data[i + 2] == 0x0d && data[i + 3] == 0x0a) {
                flow->http.data_pos = data + i + 4;
                flow->http.data_len = data_len - i - 4;
                break;
            }
            
            start = i + 2;
        }
    }
}
```

### HTTP 数据结构

```c
// app_filter.h:87-96
typedef struct http_proto {
    int match;                    // 是否匹配成功
    int method;                   // HTTP_METHOD_GET / HTTP_METHOD_POST
    char *url_pos;                // URL 位置指针
    int url_len;                  // URL 长度
    char *host_pos;               // Host 头位置指针
    int host_len;                 // Host 头长度
    char *data_pos;               // 请求体位置
    int data_len;                 // 请求体长度
} http_proto_t;
```

### HTTP 报文示例

```
GET /index.html HTTP/1.1\r\n
Host: www.example.com\r\n
User-Agent: Mozilla/5.0\r\n
\r\n
[请求体]
```

解析结果：
- `method`: GET
- `url_pos`: "/index.html"
- `host_pos`: "www.example.com"

---

## 三、HTTPS 协议解析

### 核心原理：SNI 提取（非解密）

**OpenAppFilter 不对 HTTPS 流量进行解密**，而是利用 TLS 协议的特性：

```c
// app_filter.c:679-733
int dpi_https_proto(flow_info_t *flow)
{
    char *p = flow->l4_data;
    int data_len = flow->l4_len;
    
    // 1. 检查是否是 TLS Handshake
    //    0x16 = Handshake
    //    0x03 = TLS 版本主版本号
    //    0x01 = ClientHello
    if (!((p[0] == 0x16 && p[1] == 0x03 && p[5] == 0x01) || flow->client_hello))
        return -1;
    
    // 2. 搜索 SNI 扩展特征
    //    寻找模式: 0x00 0x00 0x00 (SNI 类型标识)
    for (i = 0; i < data_len; i++) {
        if (p[i] == 0x0 && p[i + 1] == 0x0 && p[i + 2] == 0x0 && p[i + 3] != 0x0) {
            
            // 3. 提取 SNI 长度 (偏移 7)
            memcpy(&url_len, p + i + HTTPS_LEN_OFFSET, 2);
            
            if (ntohs(url_len) <= MIN_HOST_LEN || ntohs(url_len) > MAX_HOST_LEN)
                continue;
            
            // 4. 提取 SNI 主机名 (偏移 9)
            if (i + HTTPS_URL_OFFSET + ntohs(url_len) < data_len) {
                if (!check_domain(p + i + HTTPS_URL_OFFSET, ntohs(url_len)))
                    continue;
                
                flow->https.match = AF_TRUE;
                flow->https.url_pos = p + i + HTTPS_URL_OFFSET;
                flow->https.url_len = ntohs(url_len);
                flow->client_hello = 0;
                return 0;
            }
        }
    }
    
    return -1;
}
```

### HTTPS 数据结构

```c
// app_filter.h:98-102
typedef struct https_proto {
    int match;                    // 是否匹配成功
    char *url_pos;                // SNI 主机名位置指针
    int url_len;                  // SNI 主机名长度
} https_proto_t;
```

### 关键常量

```c
// app_filter.h:42-43
#define HTTPS_URL_OFFSET    9    // SNI 主机名偏移量
#define HTTPS_LEN_OFFSET    7    // SNI 长度偏移量
```

---

## 四、TLS ClientHello 报文结构

```
TLS Record Layer (5 字节):
┌─────────────────────────────────┐
│ 0x16 (Handshake)                │  ← 字节 0
├─────────────────────────────────┤
│ 0x03 0x03 (TLS 1.2)            │  ← 字节 1-2
├─────────────────────────────────┤
│ Length (2 bytes)                │  ← 字节 3-4
└─────────────────────────────────┘

Handshake Protocol:
┌─────────────────────────────────┐
│ 0x01 (ClientHello)              │  ← 字节 5
├─────────────────────────────────┤
│ Length (3 bytes)                │
├─────────────────────────────────┤
│ Client Version (2 bytes)        │
├─────────────────────────────────┤
│ Random (32 bytes)               │
├─────────────────────────────────┤
│ Session ID Length               │
│ Session ID                      │
├─────────────────────────────────┤
│ Cipher Suites Length            │
│ Cipher Suites                   │
├─────────────────────────────────┤
│ Compression Methods Length      │
│ Compression Methods             │
├─────────────────────────────────┤
│ Extensions Length (2 bytes)     │
│ Extensions:                     │
│   ┌───────────────────────────┐ │
│   │ Extension Type: 0x0000    │ │ ← SNI 扩展
│   │ Length: ...               │ │
│   │ ┌───────────────────────┐ │ │
│   │ │ Server Name List Len  │ │ │
│   │ │ ┌───────────────────┐ │ │ │
│   │ │ │ Name Type: 0      │ │ │ │ ← host_name
│   │ │ │ Name Length: N    │ │ │ │
│   │ │ │ Host Name: "..."  │ │ │ │ ← 这里提取！
│   │ │ └───────────────────┘ │ │ │
│   │ └───────────────────────┘ │ │
│   └───────────────────────────┘ │
└─────────────────────────────────┘
```

---

## 五、为什么不需要解密？

### SNI (Server Name Indication) 原理

**SNI 是 TLS 协议的扩展，在加密握手之前以明文形式发送**，原因：

1. **虚拟主机场景**：一台服务器托管多个域名
2. **证书选择**：服务器需要知道客户端想连接哪个域名，才能选择正确的证书
3. **时序要求**：SNI 必须在加密建立之前发送

### HTTPS 处理的局限性

| 能力 | 支持 | 说明 |
|------|------|------|
| ✅ 知道访问 `douyin.com` | 是 | 从 SNI 提取 |
| ❌ 知道访问 `/video/xxx` | 否 | URL 已加密 |
| ❌ 看到传输内容 | 否 | 应用数据已加密 |

---

## 六、特征匹配算法

### 1. 主匹配函数

```c
// app_filter.c:1058-1080
int match_feature(flow_info_t *flow)
{
    feature_list_read_lock();
    
    if (!list_empty(&af_feature_head)) {
        // 遍历特征链表
        list_for_each_entry_safe(node, n, &af_feature_head, head) {
            if (af_match_one(flow, node)) {
                flow->app_id = node->app_id;
                flow->feature = node;
                strncpy(flow->app_name, node->app_name, sizeof(flow->app_name) - 1);
                feature_list_read_unlock();
                return AF_TRUE;
            }
        }
    }
    
    feature_list_read_unlock();
    return AF_FALSE;
}
```

### 2. 单特征匹配顺序

```c
// app_filter.c:976-1017
int af_match_one(flow_info_t *flow, af_feature_node_t *node)
{
    // ==========================================
    // 匹配顺序：协议 → 源端口 → 目标端口 → URL → 位置特征
    // ==========================================
    
    // 1. 协议匹配
    if (node->proto > 0 && flow->l4_protocol != node->proto)
        return AF_FALSE;
    
    if (flow->l4_len == 0)
        return AF_FALSE;
    
    // 2. 源端口匹配
    if (node->sport != 0 && flow->sport != node->sport)
        return AF_FALSE;
    
    // 3. 目标端口匹配 (支持范围)
    if (!af_match_port(&node->dport_info, flow->dport))
        return AF_FALSE;
    
    // 4. URL/Host 匹配
    if (strlen(node->request_url) > 0 || strlen(node->host_url) > 0) {
        return af_match_by_url(flow, node);
    }
    // 5. 十六进制位置匹配
    else if (node->pos_num > 0) {
        return af_match_by_pos(flow, node);
    }
    // 6. 仅端口匹配也可成功
    else {
        return AF_TRUE;
    }
}
```

### 3. 端口匹配

```c
// app_filter.c:242-277
int af_match_port(port_info_t *info, int port)
{
    // 支持的格式：
    // - 单个端口: 443
    // - 端口范围: 1000-2000
    // - 排除端口: !80
    // - 多条件: 80|443|8080
    
    int i;
    int with_not = 0;
    
    if (info->num == 0)
        return 1;
    
    // 检查是否有排除条件 (!port)
    for (i = 0; i < info->num; i++) {
        if (info->range_list[i].not) {
            with_not = 1;
            break;
        }
    }
    
    // 根据是否有排除条件选择匹配逻辑
    for (i = 0; i < info->num; i++) {
        if (with_not) {
            if (info->range_list[i].not && 
                port >= info->range_list[i].start && 
                port <= info->range_list[i].end) {
                return 0;  // 排除匹配
            }
        }
        else {
            if (port >= info->range_list[i].start && 
                port <= info->range_list[i].end) {
                return 1;  // 包含匹配
            }
        }
    }
    
    return with_not ? 1 : 0;
}
```

### 4. URL 匹配

```c
// app_filter.c:930-975
int af_match_by_url(flow_info_t *flow, af_feature_node_t *node)
{
    // HTTPS: 从 SNI 提取 host
    if (flow->https.match) {
        if (strlen(node->host_url) > 0) {
            // 正则表达式匹配 SNI
            if (regexp_match(node->host_url, flow->https.url_pos)) {
                return AF_TRUE;
            }
        }
    }
    // HTTP: 从 Host 头和 URL 提取
    else if (flow->http.match) {
        if (strlen(node->host_url) > 0 && flow->http.host_pos) {
            // 匹配 Host 头
            if (regexp_match(node->host_url, flow->http.host_pos)) {
                return AF_TRUE;
            }
        }
        if (strlen(node->request_url) > 0 && flow->http.url_pos) {
            // 匹配请求 URL
            if (regexp_match(node->request_url, flow->http.url_pos)) {
                return AF_TRUE;
            }
        }
    }
    
    return AF_FALSE;
}
```

### 5. 十六进制位置匹配

```c
// app_filter.c:883-928
int af_match_by_pos(flow_info_t *flow, af_feature_node_t *node)
{
    int i;
    unsigned int pos = 0;
    
    for (i = 0; i < node->pos_num && i < MAX_POS_INFO_PER_FEATURE; i++) {
        // 支持负偏移量 (从包末尾计算)
        if (node->pos_info[i].pos < 0) {
            pos = flow->l4_len + node->pos_info[i].pos;
        }
        else {
            pos = node->pos_info[i].pos;
        }
        
        // 检查位置是否越界
        if (pos >= flow->l4_len)
            return AF_FALSE;
        
        // 检查字节值是否匹配
        if (flow->l4_data[pos] != node->pos_info[i].value)
            return AF_FALSE;
    }
    
    return AF_TRUE;
}
```

### 位置特征格式

```
00:51|01:48|02:54
│  │  │  │
│  │  │  └─ 十六进制值 0x54
│  │  └───── 分隔符
│  └──────── 十六进制值 0x48
└─────────── 偏移量 0, 1, 2...
```

负偏移量示例：
```
-1:ff  - 倒数第 1 个字节应为 0xff
```

---

## 七、HTTP vs HTTPS 处理对比

| 特性 | HTTP | HTTPS |
|------|------|-------|
| **信息来源** | HTTP 头 `Host:` | TLS ClientHello SNI |
| **主机名提取** | ✅ 完整 | ✅ 完整 |
| **URL 路径提取** | ✅ 完整 | ❌ 不可用（已加密） |
| **内容可见** | ✅ 明文 | ❌ 已加密 |
| **处理方式** | 解析 HTTP 头 | 提取 SNI 扩展 |
| **是否解密** | 不需要（已明文） | 不需要（不解密） |

---

## 八、DPI 处理流程图

```
数据包到达
    ↓
parse_flow_proto() - 解析 L3/L4
    ↓
┌─────────────────┐
│  TCP (IPPROTO_TCP) │ → dpi_http_proto()  → 提取 Host/URL
└─────────────────┘
    ↓
┌─────────────────┐
│  TCP 端口 443    │ → dpi_https_proto() → 提取 SNI
└─────────────────┘
    ↓
match_feature() - 遍历特征链表
    ↓
┌─────────────────────────────┐
│ 匹配顺序：                   │
│ 1. 协议                      │
│ 2. 源端口                    │
│ 3. 目标端口                  │
│ 4. URL/Host (HTTP Host/SNI)  │
│ 5. 十六进制位置特征           │
└─────────────────────────────┘
    ↓
匹配成功 → 记录 app_id → 执行过滤规则
```

