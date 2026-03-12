# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-12
**Commit:** 4f4ddfa
**Branch:** master

## OVERVIEW

OpenAppFilter (OAF) - OpenWrt parental control plugin for DPI-based app filtering. Filters games, videos, social media (TikTok, YouTube, Telegram, etc.) via kernel module + userspace daemon + LuCI web UI.

## STRUCTURE

```
OpenAppFilter/
├── oaf/                    # Kernel DPI module (oaf.ko)
│   └── src/                # C source for netfilter integration
├── open-app-filter/        # Userspace daemon (oafd)
│   ├── src/                # C source for ubus service
│   └── files/              # Config, init scripts, feature DB
└── luci-app-oaf/           # LuCI web interface
    ├── luasrc/             # Lua controllers and CBI models
    ├── htdocs/             # Static assets (CSS, icons)
    ├── po/                 # i18n translations
    └── root/               # uci-defaults, ACL
```

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Add app signature | `open-app-filter/files/feature_cn.cfg` | Format: `appid name proto:host:port:match` |
| Modify filtering logic | `oaf/src/app_filter.c` | Kernel DPI engine (1961 lines) |
| Add ubus API | `open-app-filter/src/appfilter_ubus.c` | 2483 lines, main RPC handler |
| Modify web UI | `luci-app-oaf/luasrc/controller/appfilter.lua` | Route definitions |
| Change config format | `open-app-filter/files/appfilter.config` | UCI format |
| User/device tracking | `open-app-filter/src/appfilter_user.c` | 1000 lines |
| Kernel-userspace comm | `oaf/src/app_filter.h:59` | Netlink ID 29 |

## CONVENTIONS

**Build System:**
- OpenWrt package Makefiles at each module root
- Kernel module: `make package/oaf/compile V=s`
- Daemon: `make package/open-app-filter/compile V=s`
- LuCI: `make package/luci-app-oaf/compile V=s`

**Code Style:**
- C: Linux kernel style for `oaf/`, OpenWrt style for `open-app-filter/`
- Lua: LuCI MVC pattern (controller → CBI model → view)
- Config: UCI format (`config section`, `option name 'value'`)

**Communication:**
- Kernel ↔ Userspace: Netlink socket (ID 29, `/proc/net/af_client`)
- Userspace ↔ Web UI: ubus (service: `appfilter`)
- Config storage: `/etc/config/appfilter`, `/etc/config/user_info`

## ANTI-PATTERNS (THIS PROJECT)

**WARNING: Aggressive compiler warning suppression in kernel module:**
```makefile
# oaf/Makefile line 26
EXTRA_CFLAGS:=-Wno-declaration-after-statement -Wno-strict-prototypes ...
```

**Unsafe string operations (potential buffer overflows):**
- `strcpy()` in `appfilter_config.c`: lines 47, 84, 123, 174, 450, 471
- `sprintf()` in `appfilter_config.c`: lines 151, 199, 223, 245, 268, 292, 309, 315, 362
- **Use `snprintf()` with `sizeof()` instead**

**Fixed pagination constant:**
```javascript
// luci-app-oaf/luasrc/view/admin_network/user_status.htm:35
// page_size is fixed at 15, DON'T update from response
```

## UNIQUE STYLES

**App Signature Format (feature.cfg v3.0):**
```
#version 6.1
appid name proto:host_match:port:request_url_match
```
Example: `1001 抖音 6:byteflow:443:sni~`

**Work Modes:**
- `0`: Gateway mode (filter enabled)
- `1`: Bypass mode (no filtering)
- `2`: Bridge mode

**Time-based filtering:**
- Daily limits per weekday (0-6)
- Time ranges with `start_time`/`end_time`
- `deny_time`/`allow_time` quotas

## COMMANDS

```bash
# Enable package in OpenWrt buildroot
echo "CONFIG_PACKAGE_luci-app-oaf=y" >> .config
make defconfig

# Compile individual components
make package/oaf/compile V=s
make package/open-app-filter/compile V=s
make package/luci-app-oaf/compile V=s

# Or compile entire firmware
make V=s

# Runtime service control
/etc/init.d/appfilter start|stop|restart
```

## NOTES

- **No CI/CD**: Project lacks automated testing/build pipeline
- **No tests**: No unit or integration tests found
- **GPL v2 licensed**
- Feature database: `/tmp/feature.cfg` (symlinked from `/etc/appfilter/feature.cfg`)
- Hardware NAT conflict: `hnat.sh` disables flow offloading for mt798x/qca chipsets
- Kernel module must be loaded before daemon starts