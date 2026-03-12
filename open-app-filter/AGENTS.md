# Open App Filter Daemon (oafd)

Userspace daemon providing ubus service for app filtering management.

## STRUCTURE

```
open-app-filter/
├── src/
│   ├── main.c              # Daemon entry (883 lines)
│   ├── appfilter_ubus.c    # UBUS RPC handlers (2483 lines)
│   ├── appfilter_user.c    # Device tracking (1000 lines)
│   ├── appfilter_config.c  # UCI config wrapper (552 lines)
│   ├── appfilter_netlink.c # Kernel communication
│   └── utils.c             # Helper functions
├── files/
│   ├── appfilter.init      # procd init (START=96)
│   ├── appfilter.config    # UCI config template
│   ├── feature_cn.cfg      # Chinese app signatures
│   ├── feature_en.cfg      # English app signatures
│   ├── hnat.sh             # Hardware NAT handler
│   └── gen_class.sh        # Classification generator
└── Makefile                # OpenWrt package
```

## WHERE TO LOOK

| Task | File | Notes |
|------|------|-------|
| Add ubus method | `src/appfilter_ubus.c` | Handler registration |
| Device tracking | `src/appfilter_user.c` | Hash table, visit info |
| Config parsing | `src/appfilter_config.c` | UCI wrappers |
| Time filtering | `src/main.c` | `af_check_time_*()` |
| Netlink to kernel | `src/appfilter_netlink.c` | Socket ID 29 |
| Init sequence | `files/appfilter.init` | procd service |

## UBUS METHODS

| Method | Purpose |
|--------|---------|
| `dev_list` | List all devices |
| `visit_list` | Visit history per device |
| `get_app_filter` / `set_app_filter` | App filter config |
| `get_app_filter_time` / `set_app_filter_time` | Time rules |
| `get_all_users` | Paginated user list |
| `add_app_filter_user` / `del_app_filter_user` | User management |
| `get_whitelist_user` / `add_whitelist_user` | Whitelist |
| `set_nickname` | Device naming |
| `class_list` | App categories |

## CONFIG FILES

| File | Purpose |
|------|---------|
| `/etc/config/appfilter` | Main UCI config |
| `/etc/config/user_info` | User settings |
| `/etc/appfilter/feature.cfg` | App signatures (symlinked to /tmp) |
| `/tmp/log/appfilter.log` | Runtime logs |

## ANTI-PATTERNS

**Unsafe string operations in `appfilter_config.c`:**
- `strcpy()`: lines 47, 84, 123, 174, 450, 471
- `sprintf()`: lines 151, 199, 223, 245, 268, 292, 309, 315, 362
- **Use `snprintf()` with `sizeof()` instead**

## NOTES

- Service name: `appfilter` (ubus)
- Log levels: `LOG_DEBUG`, `LOG_INFO`, `LOG_WARN`, `LOG_ERROR`
- Feature DB copied from `feature_cn.cfg` on first start
- `hnat.sh` disables flow offloading for mt798x/qca