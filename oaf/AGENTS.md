# OAF Kernel Module

OpenWrt kernel DPI (Deep Packet Inspection) module for app filtering via netfilter hooks.

## STRUCTURE

```
oaf/
├── src/
│   ├── app_filter.c      # Main DPI engine (1961 lines)
│   ├── af_client.c       # Client tracking (663 lines)
│   ├── af_conntrack.c    # Netfilter conntrack integration
│   ├── af_utils.c        # Kernel utilities
│   ├── af_log.c          # Logging via printk
│   ├── cJSON.c           # JSON parser (kernel-adapted)
│   └── *.h               # Headers
└── Makefile              # OpenWrt kernel package
```

## WHERE TO LOOK

| Task | File | Notes |
|------|------|-------|
| DPI engine | `src/app_filter.c` | `dpi_main()`, `match_feature()` |
| Add netfilter hook | `src/app_filter.c` | `nf_register_ops()` |
| Feature matching | `src/app_filter.c` | `af_match_one()`, `af_match_by_pos()` |
| Client tracking | `src/af_client.c` | Hash table, `/proc/net/af_client` |
| Netlink protocol | `src/app_filter.h:59` | `OAF_NETLINK_ID 29` |
| Debug/sysctl | `src/af_log.c` | `/proc/sys/oaf/*` |

## KEY TYPES/STRUCTS

```c
// app_filter.h
#define OAF_NETLINK_ID 29
#define AF_FEATURE_CONFIG_FILE "/tmp/feature.cfg"

typedef struct af_feature_node {
    u_int32_t app_id;
    char app_name[64];
    char feature[128];
    u_int32_t proto;
    port_info_t dport_info;
    char host_url[128];
    char request_url[128];
    af_pos_info_t pos_info[16];
} af_feature_node_t;

typedef struct flow_info {
    struct nf_conn *ct;
    u_int32_t src, dst;
    http_proto_t http;
    https_proto_t https;
    u_int32_t app_id;
    u_int8_t drop;
} flow_info_t;
```

## CONVENTIONS

- **Kernel style**: Linux kernel coding standards
- **Logging**: `AF_DEBUG()`, `AF_INFO()`, `AF_ERROR()` macros
- **Test mode**: `TEST_MODE()` macro, enabled via `/proc/sys/oaf/test_mode`
- **Debug level**: `/proc/sys/oaf/debug` (0-3)

## NOTES

- Aggressive warning suppression in Makefile (`EXTRA_CFLAGS`)
- Netlink magic: `0xa0b0c0d0` for message validation
- Procfs exports: `/proc/net/af_client` (JSON), `/proc/sys/oaf/*`
- Must be loaded before userspace daemon starts