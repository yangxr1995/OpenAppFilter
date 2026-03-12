# LUCI-APP-OAF KNOWLEDGE BASE

**Generated:** 2026-03-12
**Commit:** 4f4ddfa
**Branch:** master

## OVERVIEW

LuCI web interface for OpenAppFilter (OAF) - parental control plugin. Provides MVC-based UI for managing app filtering rules, users, time limits, and device settings.

## STRUCTURE

```
luci-app-oaf/
├── luasrc/
│   ├── controller/appfilter.lua    # 573 lines - routes, ubus calls
│   ├── model/cbi/appfilter/*.lua   # 8 CBI models (forms)
│   └── view/admin_network/*.htm    # Template views
├── htdocs/luci-static/resources/   # CSS, icons
├── po/zh_Hans/                      # i18n translations
└── root/usr/share/rpcd/acl.d/      # ACL permissions
```

## WHERE TO LOOK

| Task | Location |
|------|----------|
| Add/modify route | `luasrc/controller/appfilter.lua` |
| Modify form/model | `luasrc/model/cbi/appfilter/<name>.lua` |
| Change UI template | `luasrc/view/admin_network/*.htm` |
| Update translations | `po/zh_Hans/appfilter.po` |
| Adjust ACL | `root/usr/share/rpcd/acl.d/appfilter.json` |

## ROUTES

UI routes under `/admin/services/appfilter/`:
- `user_list` - Main dashboard (users + devices)
- `app_filter` - App filtering rules
- `time` - Time configuration
- `user` - User settings
- `feature` - App feature library
- `advance` - Advanced settings

JSON API routes under `/admin/network/`:
- `user_status`, `get_user_list`, `dev_visit_list`
- `set_app_filter`, `get_app_filter`, `get_app_filter_base`
- `set_app_filter_user`, `add/del_app_filter_user`
- `add/del_whitelist_user`, `feature_upgrade`
- And 20+ more ubus-proxied endpoints

## CBI MODELS

8 models in `luasrc/model/cbi/appfilter/`:
- `user_list.lua` - User management
- `dev_status.lua` - Device status display
- `app_filter.lua` - App selection form
- `time.lua` - Time settings
- `user.lua` - User configuration
- `feature.lua` - Feature library management
- `advance.lua` - Advanced options
- `time_setting.lua` - Detailed time configuration

## NOTES

- All CBI models use `{hideapplybtn=true, hidesavebtn=true, hideresetbtn=true}`
- Pagination in `user_status.htm` uses fixed `page_size=15` (do not override from response)
- Controller proxies most calls to ubus service `appfilter`
- Logs to `/tmp/log/oaf_luci.log` via `llog()` function
