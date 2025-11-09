# 设备接口与指令对照表

本文件逐条列出了前端项目中出现的所有网络请求，包括请求方式、URL、HTTP 头、查询/表单参数及其含义，便于在 Apifox 等调试工具中完整复现。若无特殊说明，所有接口都基于 `import.meta.env.VITE_BASE_URL`（默认 `http://192.168.0.1`）。

## 1. 全局配置

### 1.1 路径与公共请求头
- **基础路径**：
  - 普通机型：`{BASE}/goform/goform_get_cmd_process`（GET）、`{BASE}/goform/goform_set_cmd_process`（POST）。
  - R186X 机型：`{BASE}/reqproc/proc_get`（GET）、`{BASE}/reqproc/proc_post`（POST）。
  - 两者会根据 `public/serverConfig.json` 中的 `is_r186x` 动态切换。【F:src/api/user.ts†L9-L92】
- **公共请求头**（GET、POST 共用，可配置在 Apifox 环境级 Header）：
  - `Accept: application/json, text/javascript, */*; q=0.01`
  - `Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6`
  - `Content-Type: application/x-www-form-urlencoded; charset=UTF-8`
  以上由 `getapi`/`postapi` 封装统一注入。【F:src/api/user.ts†L66-L142】

### 1.2 通用封装
- `httpapi(method, apiUrl, head, body)`：对外开放的自由请求入口，校验方法、解析自定义 Header 与 Body 后直接调用 Axios。【F:src/api/user.ts†L18-L52】
- `getapi(cmd, hide)`：拼接查询串（追加 `_=时间戳`，`hide=true` 时添加 `hide=Young`）并沿用公共 Header。【F:src/api/user.ts†L77-L142】
- `postapi(data)`：以 `application/x-www-form-urlencoded` 发送表单数据，可接受普通对象或 `URLSearchParams`。【F:src/api/user.ts†L66-L142】
- `setnet(goformId)`：提交 `URLSearchParams`，固定携带 `notCallback=true` 与传入的 `goformId`，用于断网/拨号等操作。【F:src/api/user.ts†L144-L157】
- `outLogin()`：快捷登出封装，内部调用 `postapi({ goformId: 'LOGOUT' })`。【F:src/api/user.ts†L94-L105】

## 2. GET 查询指令
所有 GET 请求都沿用 [1.1](#11-路径与公共请求头) 的 Header，并在查询串末尾附加 `_=时间戳` 防止缓存；若页面希望静默刷新，会额外添加 `hide=Young` 关闭进度条提示。

### 2.1 会话与概览

#### 2.1.1 登录态初始化（`islogin`）
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`：请求服务端一次返回多个字段。
  - `cmd=modem_main_state,pin_status,blc_wan_mode,blc_wan_auto_mode,loginfo,fota_new_version_state,fota_current_upgrade_state,fota_upgrade_selector,network_provider,is_mandatory,sta_count,m_sta_count`：组合指令；各字段含义：
    - `modem_main_state`：调制解调器当前状态。
    - `pin_status`：SIM PIN 校验结果。
    - `blc_wan_mode` / `blc_wan_auto_mode`：WAN 拨号模式及是否自动。
    - `loginfo`：登录状态，`ok` 表示已登录。
    - `fota_*`：固件升级最新版本、当前状态、升级策略。
    - `network_provider`：当前运营商名称。
    - `is_mandatory`：是否存在强制升级/操作。
    - `sta_count` / `m_sta_count`：主/访客 Wi-Fi 在线终端数量。【F:src/api/user.ts†L107-L117】

#### 2.1.2 仪表盘心跳轮询
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`：批量返回指标。
  - `sms_received_flag_flag=0`、`sts_received_flag_flag=0`：清零短信/USSD 提醒标志后再取数。
  - `cmd={50+ 指令}`：完整字符串见源码；常用字段含义：
    - `signalbar,network_type,sub_network_type,rssi,rscp,lte_rsrp`：信号强度与制式。
    - `ppp_status,internet_status`：拨号与联网状态。
    - `EX_SSID1,EX_wifi_profile,m_ssid_enable`：访客网络配置。
    - `wifi_cur_state,SSID1,ssid,show_ssid_on_lcd`：主 Wi-Fi 状态与展示。
    - `simcard_roam,roam_setting_option,upg_roam_switch`：漫游状态与策略。
    - `lan_ipaddr,default_wan_name,ethwan_mode`：LAN IP、默认 WAN、是否启用以太网口。
    - `battery_charging,battery_vol_percent,battery_pers`：电池电量信息。
    - `spn_name_data,spn_b1_flag,spn_b2_flag`：SIM SPN 显示字段。
    - `realtime_*`、`monthly_*`：实时与当月流量/速率统计。
    - `traffic_alined_delta,date_month`：套餐统计偏移与月份。
    - `data_volume_limit_switch,data_volume_limit_size,data_volume_alert_percent,data_volume_limit_unit`：流量阈值配置。
    - `fota_package_already_download`：固件包下载状态。
    - `vpn_state,connect_status`：VPN 连接状态。
    - `sms_received_flag,sts_received_flag,sms_unread_num`：短信/USSD 未读数量。【F:src/main.ts†L223-L269】

#### 2.1.3 信号刷新
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`：批量获取信号字段。
  - `cmd=network_type,sub_network_type,rssi,rscp,lte_rsrp`：制式与信号强度指标。【F:src/main.ts†L244-L251】

#### 2.1.4 设备概览
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`：批量返回。
  - `cmd=wifi_coverage,...,detail_cell_id`：包含 Wi-Fi 配置、SIM/蜂窝信息、LAN/WAN 地址以及小区指标；各字段按名称对应页面展示内容。【F:src/main.ts†L256-L264】

#### 2.1.5 短信汇总
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `cmd=sms_data_total`：读取短信列表统计。
  - `page=0`：第一页。
  - `data_per_page=500`：一次拉取 500 条。
  - `mem_store=1`：访问设备 NV 存储。
  - `tags=10`：组合收/发/草稿箱。
  - `order_by=order by id desc`：按 ID 倒序。【F:src/main.ts†L266-L269】

#### 2.1.6 短信容量
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：`cmd=sms_capacity_info`，返回设备短信存储容量字段。【F:src/views/page/connect/sms.vue†L23-L44】

#### 2.1.7 云端通知
- **Method & URL**：`GET https://wifi.api.my-youth.cn/`。
- **Query 参数**：`version={appVersion}`，用于告知云端当前前端版本，返回公告与模板信息。【F:src/api/user.ts†L171-L175】

#### 2.1.8 飞猫设备信息
- **Method & URL**：`GET http://{ip}:8081/fm_get_device_information`。
- **Headers**：`Accept: application/json`（接口返回 JSON）。
- **Query 参数**：无。
- **说明**：读取飞猫 8081 端口的扩展信息。【F:src/api/user.ts†L177-L190】

### 2.2 终端与 Wi-Fi

#### 2.2.1 有线终端列表
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `cmd=lan_station_list`：请求有线终端信息。
  - `hide=Young`：静默刷新不显示加载条。
  - `_=时间戳`：避免缓存。
- **说明**：返回 `lan_station_list` 数组。【F:src/api/user.ts†L119-L129】

#### 2.2.2 无线终端列表
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `cmd=station_list`：请求 Wi-Fi 终端列表。
  - `hide=Young`：静默刷新。
  - `_=时间戳`：避免缓存。
- **说明**：返回 `station_list` 数组，连接列表页也会直接调用（可不带 `hide`）。【F:src/api/user.ts†L131-L142】【F:src/views/page/connect/list.vue†L1-L40】

#### 2.2.3 Wi-Fi 性能参数
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`：批量返回。
  - `cmd=WirelessMode,wifi_band,CountryCode,MAX_Access_num,m_MAX_Access_num,Channel,wifi_11n_cap,wifi_sta_connection`：获取主/访客 Wi-Fi 最大连接数、信道、11n 能力与当前连接数量。【F:src/views/page/connect/WifiBand.vue†L1-L54】

#### 2.2.4 Wi-Fi 快捷配置草稿
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `cmd=sms_data_total`：读取短信草稿。
  - `page=0`、`data_per_page=500`、`mem_store=1`：同 2.1.5。
  - `tags=4`：限定草稿箱。
  - `number=Config`：匹配自定义 HTTP 配置短信。
  - `order_by=order by id desc`：倒序排列。
- **说明**：欢迎页与“自定义 HTTP API”设置共用的配置读取逻辑。【F:src/views/welcome/index.vue†L220-L252】【F:src/views/page/SimCard/HttpAPI.vue†L28-L76】

### 2.3 网络与 LAN

#### 2.3.1 LAN/DHCP 基础参数
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`。
  - `cmd=lan_ipaddr,lan_netmask,mac_address,dhcpEnabled,dhcpStart,dhcpEnd,dhcpLease_hour`：读取 LAN IP、子网掩码、DHCP 范围与租期。【F:src/views/page/connect/route.vue†L26-L41】

#### 2.3.2 流量告警配置
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`。
  - `cmd=data_volume_alert_percent,data_volume_limit_size,data_volume_limit_unit,monthly_tx_bytes,monthly_rx_bytes,monthly_time,ppp_status`：读取流量阈值与当月统计。【F:src/layout/components/notice/index.vue†L33-L74】

#### 2.3.3 VPN 参数
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`。
  - `cmd=vpn_name,vpn_password,vpn_server_ip,vpn_state,vpn_type,vpn_mode,connect_status,vpn_status`：读取 VPN 配置、类型、连接状态。【F:src/views/page/QuickSettings/VPN.vue†L16-L36】

#### 2.3.4 APN 列表
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`。
  - `cmd=APN_config0,...,APN_config19,ipv6_APN_config0,...,ipv6_APN_config19,m_profile_name,profile_name,wan_dial,pdp_type,pdp_select,index,Current_index,apn_auto_config,ipv6_apn_auto_config,apn_mode,wan_apn,ppp_auth_mode,ppp_username,ppp_passwd,ipv6_wan_apn,ipv6_pdp_type,ipv6_ppp_auth_mode,ipv6_ppp_username,ipv6_ppp_passwd,apn_num_preset`：一次获取全部 IPv4/IPv6 APN 预设、当前拨号信息与认证参数。【F:src/views/page/QuickSettings/APN.vue†L95-L133】

### 2.4 高级设置

#### 2.4.1 频段列表
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：`cmd=set_band_list` 或 `cmd=get_support_band`，分别返回当前频段选择与设备支持的频段列表。【F:src/views/page/AdvancedSetting/Band.vue†L31-L53】

#### 2.4.2 R186X 频段状态
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `cmd=lte_band,cell_id,ping_google`、`multi_data=1`：查询当前驻留频段与小区信息。
  - `cmd=work_lte_band`、`multi_data=1`：获取工作频段掩码。
- **说明**：仅在 R186X 机型中调用。【F:src/views/page/AdvancedSetting/Band.vue†L139-L170】

#### 2.4.3 DDNS 配置
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：`multi_data=1&cmd=DDNS_Enable,DDNS_Mode,DDNSProvider,DDNSAccount,DDNSPassword,DDNS,DDNS_Hash_Value`，返回当前 DDNS 设置信息。【F:src/views/page/AdvancedSetting/DDNS.vue†L20-L42】

#### 2.4.4 TR069 配置
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：`multi_data=1&cmd=tr069_enable,tr069_acs_username,tr069_acs_password,tr069_acs_url,tr069_inform_enable,tr069_inform_interval,tr069_cpe_auth_enable,tr069_cpe_username,tr069_cpe_password`，读取远程管理参数。【F:src/views/page/AdvancedSetting/TR069.vue†L20-L39】

#### 2.4.5 SNTP 时间服务器
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `multi_data=1`。
  - `cmd=sntp_year,sntp_month,sntp_day,sntp_hour,sntp_minute,sntp_second,sntp_time_set_mode,sntp_static_server0,sntp_static_server1,sntp_static_server2,sntp_server0,sntp_server1,sntp_server2,sntp_server3,sntp_server4,sntp_server5,sntp_server6,sntp_server7,sntp_server8,sntp_server9,sntp_other_server0,sntp_other_server1,sntp_other_server2,sntp_timezone,sntp_timezone_index,sntp_dst_enable,sntp_process_result`：读取手动时间与服务器列表。【F:src/views/page/AdvancedSetting/Other.vue†L43-L65】

#### 2.4.6 快速开机/密码状态
- **Method & URL**：`GET {BASE}/{getpath('get')}`。
- **Query 参数**：
  - `cmd=mgmt_quicken_power_on,need_hard_reboot,need_sim_pin&multi_data=1`：读取快速开机、硬重启、SIM PIN 状态。
  - `cmd=current_Password,admin_Password,root_Password&multi_data=1`：读取设备当前/管理员/超级管理员密码（加密后）。【F:src/views/page/AdvancedSetting/Other.vue†L66-L79】

## 3. POST `goformId` 指令
除特别说明外，以下请求均向 `POST {BASE}/{getpath('post')}` 发送 `application/x-www-form-urlencoded` 表单，并沿用 [1.1](#11-路径与公共请求头) 的 Header。

### 3.1 认证与系统

#### 3.1.1 登录
- **Body 参数**：
  - `goformId=LOGIN`：指定登录操作。
  - `username=admin`：固定用户名。
  - `password={Base64 明文}`：前端对用户输入的明文密码做 Base64 后提交。【F:src/views/login/index.vue†L44-L83】

#### 3.1.2 登出
- **Body 参数**：`goformId=LOGOUT`，无额外字段。【F:src/api/user.ts†L94-L105】

#### 3.1.3 修改管理员密码
- **Body 参数**：
  - `goformId=CHANGE_PASSWORD`。
  - `oldPassword`：旧密码明文。
  - `newPassword`：新密码明文。【F:src/layout/components/navbar.vue†L193-L216】

#### 3.1.4 SNTP 时间同步
- **Body 参数**：
  - `goformId=SNTP`：提交时间服务器配置。
  - `manualsettime`：`auto`/`manual`，区分自动或手动。
  - `sntp_server{1-3}_ip`：最多三个服务器地址。
  - `timezone`、`sntp_timezone_index`：时区字符串及索引。
  - `DaylightEnabled`：是否启用夏令时。
  - `time_year`、`time_month`、`time_day`、`time_hour`、`time_minute`：手动校时用的时间字段。【F:src/views/page/AdvancedSetting/Other.vue†L121-L170】

#### 3.1.5 快速开机关闭
- **Body 参数**：
  - `goformId=MGMT_CONTROL_POWER_ON_SPEED`。
  - `mgmt_quicken_power_on=1|0`：开启或关闭快速开机。【F:src/views/page/AdvancedSetting/Other.vue†L80-L114】

#### 3.1.6 设备操作命令
- **Body 参数**：`goformId` 取以下值之一：`REBOOT_DEVICE`（重启）、`TURN_OFF_DEVICE`（关机）、`RESTORE_FACTORY_SETTINGS`（恢复出厂）；无需额外字段，界面会先弹出确认提示。【F:src/views/page/AdvancedSetting/Other.vue†L210-L357】

#### 3.1.7 ADB 模式切换
- **Body 参数**：
  - `goformId=SET_DEVICE_MODE`。
  - `device_mode`：`adb`/`normal` 等枚举，表示 ADB 是否开启。
  - `need_reboot`：`1` 表示需要后续重启。
  - 触发后如需重启会再调用 `REBOOT_DEVICE`。【F:src/views/page/AdvancedSetting/Other.vue†L210-L357】

#### 3.1.8 修改 USB MAC
- **Body 参数**：
  - `goformId=SET_USB_MAC_ADDRESS`。
  - `mac`：新的 MAC 地址字符串。【F:src/views/page/AdvancedSetting/Other.vue†L268-L285】

#### 3.1.9 修改 IMEI
- **Body 参数**：
  - `goformId=SET_IMEI_NUM`。
  - `imei`：15 位新 IMEI。【F:src/views/page/AdvancedSetting/Other.vue†L286-L305】

#### 3.1.10 执行 AT 指令
- **Body 参数**：
  - `goformId=EXECUTE_AT_COMMAND`。
  - `at_cmd`：要执行的 AT 字符串。【F:src/views/page/AdvancedSetting/AT.vue†L18-L60】

### 3.2 Wi-Fi 与网络

#### 3.2.1 主 Wi-Fi 开关
- **Body 参数**：
  - `goformId=SET_WIFI_INFO`。
  - `wifiEnabled=1|0`：开启或关闭主 Wi-Fi。【F:src/views/welcome/index.vue†L168-L191】【F:src/views/page/QuickSettings/index.vue†L45-L89】

#### 3.2.2 Wi-Fi 覆盖范围
- **Body 参数**：
  - `goformId=SET_WIFI_COVERAGE`。
  - `wifi_coverage`：功率档位（如 `high`/`middle`/`low`）。【F:src/views/page/connect/index.vue†L56-L86】

#### 3.2.3 Wi-Fi 休眠
- **Body 参数**：
  - `goformId=SET_WIFI_SLEEP_INFO`。
  - `sysIdleTimeToSleep`：休眠前空闲分钟数。【F:src/views/page/connect/index.vue†L56-L94】

#### 3.2.4 Wi-Fi 频段与信道
- **Body 参数**：
  - `goformId=SET_WIFI_INFO`。
  - `wifiMode`：无线模式（如 `802.11bgn`）。
  - `countryCode`：地区码。
  - `MAX_Access_num` / `m_MAX_Access_num`：主/访客最大连接数。
  - `wifi_band`：`b`(2.4G)/`a`(5G)/`ab`(双频)。
  - `selectedChannel`：信道号。
  - `abg_rate`、`wifi_11n_cap`：速率与 11n 功能。
  - 其它字段按界面映射自动填充。【F:src/views/page/connect/WifiBand.vue†L26-L58】

#### 3.2.5 LAN/DHCP 设置
- **Body 参数**：
  - `goformId=DHCP_SETTING`。
  - `lanIp`、`lanNetmask`：LAN 地址与子网掩码。
  - `lanDhcpType=ENABLE|DISABLE`：是否启用 DHCP。
  - `dhcpStart`、`dhcpEnd`：DHCP 地址池起止（开启时必填）。
  - `dhcpLease`：租期小时数（开启时必填）。
  - `dhcp_reboot_flag=1`：提示需要重启生效。【F:src/views/page/connect/route.vue†L54-L116】

#### 3.2.6 VPN 设置
- **Body 参数**：
  - `goformId=GOFORM_OPEN_VPN`。
  - `vpn_name`：配置名称。
  - `vpn_password`：VPN 密码。
  - `vpn_server_ip`：服务器地址。
  - `vpn_type` / `vpn_mode`：VPN 类型（PPTP/L2TP 等）与工作模式。
  - 其他字段由后端填充或沿用旧值。【F:src/views/page/QuickSettings/VPN.vue†L37-L72】

#### 3.2.7 频段设置（普通机型）
- **Body 参数**：
  - `goformId=GOFORM_SET_BAND`。
  - `band_list`：逗号分隔的频段代号集合，提交后需手动重启。【F:src/views/page/AdvancedSetting/Band.vue†L96-L133】

#### 3.2.8 频段设置（R186X）
- **Body 参数**：
  - `goformId=SET_FREQ_BAND`。
  - `work_lte_band`：8 组逗号分隔的十进制掩码。
  - `ping_google=no`：绕过连通性检测。
  - 其他字段沿用当前状态。【F:src/views/page/AdvancedSetting/Band.vue†L139-L179】

### 3.3 数据套餐与流量

#### 3.3.1 流量阈值配置
- **Body 参数**：
  - `goformId=DATA_LIMIT_SETTING`。
  - `data_volume_limit_switch=0|1`：开关流量限制。
  - `data_volume_limit_unit=data|time`：按流量或时长计量。
  - `data_volume_alert_percent`：提醒百分比。
  - `data_volume_limit_size`：阈值（前端会按单位转换为字节/小时）。【F:src/views/page/QuickSettings/DataPlan.vue†L200-L225】

#### 3.3.2 手动校准用量
- **Body 参数**：
  - `goformId=FLOW_CALIBRATION_MANUAL`。
  - `calibration_way=data|time`：按流量或时长校准。
  - `data`：新的流量字节数（`calibration_way=data` 时必填）。
  - `time`：新的时长小时数（`calibration_way=time` 时必填）。【F:src/views/page/QuickSettings/DataPlan.vue†L232-L260】

### 3.4 APN 与拨号

#### 3.4.1 断开网络
- **Body 参数**：
  - `goformId=DISCONNECT_NETWORK`。
  - `notCallback='DISCONNECT_NETWORK'`：标记来源，避免后台回调重复执行。【F:src/views/page/QuickSettings/APN.vue†L198-L223】

#### 3.4.2 设置默认 APN
- **Body 参数**：
  - `goformId=APN_PROC_EX`。
  - `apn_action=set_default`：设为默认。
  - `apn_mode=manual`：使用手动 APN。
  - `set_default_flag=manual`：配合前端逻辑标记。
  - `pdp_type`：拨号类型，来自当前配置。
  - `index`：要设为默认的配置索引。
- **说明**：操作完成后会再次调用 `CONNECT_NETWORK` 重新拨号。【F:src/views/page/QuickSettings/APN.vue†L198-L232】

#### 3.4.3 重新拨号
- **Body 参数**：
  - `goformId=CONNECT_NETWORK`。
  - `notCallback=true`：前端无需等待后台回调。【F:src/views/page/QuickSettings/APN.vue†L226-L233】

#### 3.4.4 删除手动 APN
- **Body 参数**：
  - `goformId=APN_PROC_EX`。
  - `apn_action=delete`。
  - `apn_mode=manual`。
  - `index`：要删除的配置索引。【F:src/views/page/QuickSettings/APN.vue†L270-L298】

#### 3.4.5 切换至自动 APN
- **Body 参数**：
  - `goformId=APN_PROC_EX`。
  - `apn_mode=auto`：切换为自动配置。【F:src/views/page/QuickSettings/APN.vue†L299-L308】

#### 3.4.6 新增/保存 APN
- **Body 参数**：
  - `goformId=APN_PROC_EX`。
  - `apn_action=save`：保存手动配置。
  - `profile_name`：APN 名称。
  - `wan_apn`：拨号接入点名称。
  - `ppp_auth_mode`：认证方式（`NONE`/`PAP`/`CHAP`）。
  - `ppp_username` / `ppp_passwd`：账号与密码。
  - `ipv6_wan_apn`、`ipv6_ppp_*`：IPv6 对应字段（按需提交）。
  - 其它字段（如 `index`、`apn_mode`）根据页面选择动态组合。
- **说明**：保存后通常会调用 `CONNECT_NETWORK` 使配置生效。【F:src/views/page/QuickSettings/APN.vue†L321-L370】

### 3.5 短信与通知

#### 3.5.1 删除短信
- **Body 参数**：
  - `goformId=DELETE_SMS`。
  - `msg_id={id1;id2;}`：以分号结尾的短信 ID 列表。
  - `notCallback=true`：无需后台回调。【F:src/views/page/connect/sms.vue†L45-L72】【F:src/layout/components/notice/index.vue†L90-L119】

#### 3.5.2 标记已读
- **Body 参数**：
  - `goformId=SET_MSG_READ`。
  - `tag=0`：将消息标记为已读。
  - `msg_id={id;}`：待处理短信 ID。【F:src/views/page/connect/sms.vue†L73-L98】

#### 3.5.3 保存草稿
- **Body 参数**：
  - `goformId=SAVE_SMS`。
  - `Index=-1`：表示新增草稿。
  - `draft_group_id=''`：草稿组 ID（保持空）。
  - `notCallback=true`：无需后台回调。
  - `SMSNumber`：收件人号码或配置关键字（如 `Config;`）。
  - `encode_type=GSM7_default|UNICODE`：短信编码类型。
  - `sms_time`：前端生成的发送时间戳。
  - `SMSMessage`：经 `asciiToHex` 编码的短信内容。【F:src/views/page/connect/sms.vue†L99-L132】【F:src/views/page/SimCard/HttpAPI.vue†L59-L120】

#### 3.5.4 发送短信
- **Body 参数**：
  - `goformId=SEND_SMS`。
  - `ID=-1`：新短信。
  - 其余字段与保存草稿相同（`notCallback`、`SMSNumber`、`encode_type`、`sms_time`、`SMSMessage`）。【F:src/views/page/connect/sms.vue†L133-L177】

### 3.6 DDNS 与 TR069

#### 3.6.1 DDNS 保存
- **Body 参数**：
  - `goformId=DDNS`。
  - `DDNS_Enable=0|1`：开关。
  - `DDNS_Mode=auto|manual`：自动或手动。
  - `DDNSProvider`：服务商。
  - `DDNS`：域名。
  - `DDNSAccount` / `DDNSPassword`：账号与密码。
  - `DDNS_Hash_Value`：哈希校验值。【F:src/views/page/AdvancedSetting/DDNS.vue†L43-L76】

#### 3.6.2 TR069 保存
- **Body 参数**：
  - `goformId=GORORM_SET_TR069`。
  - `tr069enanble=0|1`：开关。
  - `tr069AcsName` / `tr069AcsPassword`：ACS 认证。
  - `tr069AcsAddress`：ACS 地址。
  - `tr069InformEnable=0|1`：是否上报。
  - `tr069InformInterval`：上报间隔秒数。
  - `tr069AuthEnable=0|1`：是否启用 CPE 认证。
  - `tr069CpeName` / `tr069CpePassword`：CPE 用户名密码。【F:src/views/page/AdvancedSetting/TR069.vue†L40-L80】

## 4. 其它接口
- **自定义 HTTP 调试**：前端 `httpapi` 界面允许输入任意 URL、方法与自定义 Header/Body，并把配置以 `SAVE_SMS` 草稿形式保存，读取时调用 2.2.4 所述查询。重现时可直接在 Apifox 中手动填写相同字段。【F:src/views/page/SimCard/HttpAPI.vue†L14-L206】【F:src/views/page/SimCard/HttpAPI.vue†L207-L280】
- **飞猫设备参数设置**：`POST http://{ip}:8081/fm_set_device_parameters`，Headers 为 `Accept: application/json`、`Content-Type: application/json`，Body 结构取决于要调整的参数（如卡槽、APN 等）。【F:src/api/user.ts†L177-L190】

