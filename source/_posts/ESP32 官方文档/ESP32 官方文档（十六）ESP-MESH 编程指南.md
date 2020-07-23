---
title: ESP32 官方文档（十六）ESP-MESH 编程指南
date: 2019-03-02 19:03:45
categories:
- ESP32 官方文档
tags:
- ESP32
---

# ESP-MESH 编程指南

这是 ESP-MESH 的编程指南，包括 API 参考和编码示例。本指南分为以下几部分：

 - ESP-MESH 编程模型
 - 编写 ESP-MESH 应用程序
 - 自组织网络
 - 应用实例
 - API 参考

有关 ESP-MESH 协议的文档，请参阅 [ESP-MESH API 指南](https://mp.csdn.net/postedit/86743079)。

<!--more-->

## ESP-MESH 编程模型

### 软件栈

ESP-MESH 软件栈构建在 Wi-Fi 驱动/FreeRTOS 之上，并且在某些情况下可以使用 LwIP 栈（即根节点）。下图说明了 ESP-MESH 软件栈。
![ESP-MESH 软件栈](https://img-blog.csdnimg.cn/2019030215595528.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
### 系统事件

应用程序通过 ESP-MESH 事件与 ESP-MESH 交互。由于 ESP-MESH 构建在 Wi-Fi 栈的顶部，因此应用程序也可以通过 Wi-Fi 事件任务与 Wi-Fi 驱动交互。下图说明了 ESP-MESH 应用程序中各种系统事件的接口。
![ESP-MESH 系统事件](https://img-blog.csdnimg.cn/20190302162721734.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
`mesh_event_id_t` 定义所有可能的  ESP-MESH 系统事件，并且可以指示诸如父/子的连接/断开之类的事件。在可以使用 ESP-MESH 系统事件之前，应用程序必须通过 `esp_mesh_set_config()` 注册 Mesh 事件回调。 回调用于从 ESP-MESH 栈以及 LwIP 栈接收事件，并且应包含与应用程序相关的每个事件的处理程序。

系统事件的典型用例包括使用诸如 `MESH_EVENT_PARENT_CONNECTED` 和 `MESH_EVENT_CHILD_CONNECTED` 之类的事件来指示节点何时可以分别开始上游和下游传输数据。 同样，`MESH_EVENT_ROOT_GOT_IP` 和 `MESH_EVENT_ROOT_LOST_IP` 可用于指示根节点何时能够和不能将数据传输到外部 IP 网络。

>在自组织模式下使用 ESP-MESH 时，用户必须确保不会调用 Wi-Fi API。这是因为自组织模式将在内部进行 Wi-Fi API 调用以连接/断开/扫描等。来自应用程序的任何 Wi-Fi 调用（包括来自回调函数和 Wi-Fi 事件处理程序的调用）可能干扰 ESP-MESH 的自组织行为。因此，在调用 `esp_mesh_start()` 之后，并且在调用 `esp_mesh_stop()` 之前，用户不应该调用 Wi-Fi API。

### LwIP & ESP-MESH

**应用程序可以直接访问 ESP-MESH 栈，而无需通过 LwIP 栈。仅根节点需要 LwIP 栈向/从外部 IP 网络发送/接收数据。** 但是，由于每个节点都可能成为根节点（由于自动根节点选择），每个节点仍必须初始化 LwIP 栈。

**每个节点都需要通过调用 `tcpip_adapter_init()` 来初始化 LwIP 栈。为了防止非根节点访问 LwIP 栈，应用程序应在 LwIP 栈初始化后停止以下服务**：

 - SoftAP 接口上的 DHCP 服务器服务。
 - Station 接口上的 DHCP 客户端服务。

以下代码段演示了如何为 ESP-MESH 应用程序初始化 LwIP。

```
/*  tcpip initialization */
tcpip_adapter_init();
/*
 * for mesh
 * stop DHCP server on softAP interface by default
 * stop DHCP client on station interface by default
 */
ESP_ERROR_CHECK(tcpip_adapter_dhcps_stop(TCPIP_ADAPTER_IF_AP));
ESP_ERROR_CHECK(tcpip_adapter_dhcpc_stop(TCPIP_ADAPTER_IF_STA));
/* do not specify system event callback, use NULL instead. */
ESP_ERROR_CHECK(esp_event_loop_init(NULL, NULL));
```

>ESP-MESH 要求根节点与路由器连接。因此，如果节点成为根节点，则相应的处理程序必须启动 DHCP 客户端服务并立即获取 IP 地址。这样做将允许其他节点开始向/从外部 IP 网络发送/接收分组。但是，如果使用静态 IP 设置，则不需要此步骤。

## 编写 ESP-MESH 应用程序

**启动 ESP-MESH 的先决条件是初始化 LwIP 和 Wi-Fi。** 以下代码片段演示了初始化 ESP-MESH 之前必要的先决条件步骤。

```
tcpip_adapter_init();
/*
 * for mesh
 * stop DHCP server on softAP interface by default
 * stop DHCP client on station interface by default
 */
ESP_ERROR_CHECK(tcpip_adapter_dhcps_stop(TCPIP_ADAPTER_IF_AP));
ESP_ERROR_CHECK(tcpip_adapter_dhcpc_stop(TCPIP_ADAPTER_IF_STA));
/* do not specify system event callback, use NULL instead. */
ESP_ERROR_CHECK(esp_event_loop_init(NULL, NULL));

/*  Wi-Fi initialization */
wifi_init_config_t config = WIFI_INIT_CONFIG_DEFAULT();
ESP_ERROR_CHECK(esp_wifi_init(&config));
ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_FLASH));
ESP_ERROR_CHECK(esp_wifi_start());
```

初始化 LwIP 和 Wi-Fi 后，启动和运行 ESP-MESH 网络的过程可归纳为以下三个步骤：

 - 初始化 Mesh
 - 配置 ESP-MESH 网络
 - 启动 Mesh

### 初始化 Mesh

以下代码段演示了如何初始化 ESP-MESH

```
/*  mesh initialization */
ESP_ERROR_CHECK(esp_mesh_init());
```

### 配置 ESP-MESH 网络

**ESP-MESH 通过 `esp_mesh_set_config()` 配置，它使用 `mesh_cfg_t` 结构接收其参数。** 该结构包含用于配置 ESP-MESH 的以下参数：

| Parameter | Description |
|----|----|
| Channel | Range from 1 to 14 |
| Event Callback | Callback for Mesh Events, see mesh_event_cb_t |
| Mesh ID | ID of ESP-MESH Network, see mesh_addr_t |
| Router | Router Configuration, see mesh_router_t |
| Mesh AP | Mesh AP Configuration, see mesh_ap_cfg_t |
| Crypto Functions | Crypto Functions for Mesh IE, see mesh_crypto_funcs_t |

以下代码段演示了如何配置 ESP-MESH。

```
/* Enable the Mesh IE encryption by default */
mesh_cfg_t cfg = MESH_INIT_CONFIG_DEFAULT();
/* mesh ID */
memcpy((uint8_t *) &cfg.mesh_id, MESH_ID, 6);
/* mesh event callback */
cfg.event_cb = &mesh_event_handler;
/* channel (must match the router's channel) */
cfg.channel = CONFIG_MESH_CHANNEL;
/* router */
cfg.router.ssid_len = strlen(CONFIG_MESH_ROUTER_SSID);
memcpy((uint8_t *) &cfg.router.ssid, CONFIG_MESH_ROUTER_SSID, cfg.router.ssid_len);
memcpy((uint8_t *) &cfg.router.password, CONFIG_MESH_ROUTER_PASSWD,
       strlen(CONFIG_MESH_ROUTER_PASSWD));
/* mesh softAP */
cfg.mesh_ap.max_connection = CONFIG_MESH_AP_CONNECTIONS;
memcpy((uint8_t *) &cfg.mesh_ap.password, CONFIG_MESH_AP_PASSWD,
       strlen(CONFIG_MESH_AP_PASSWD));
ESP_ERROR_CHECK(esp_mesh_set_config(&cfg));
```

### 启动 Mesh

以下代码段演示了如何启动 ESP-MESH。

```
/* mesh start */
ESP_ERROR_CHECK(esp_mesh_start());
```

**启动 ESP-MESH 后，应用程序应检查 ESP-MESH 事件以确定它何时连接到网络。连接后，应用程序可以使用 `esp_mesh_send()` 和 `esp_mesh_recv()` 通过 ESP-MESH 网络开始发送和接收数据包。**

## 自组织网络

自组织网络是 ESP-MESH 的一项功能，节点可以自动扫描/选择/连接/重新连接到其他节点和路由器。此功能允许 ESP-MESH 网络通过使网络对动态网络拓扑和条件具有鲁棒性来实现高度自治。启用自组织网络后，ESP-MESH 网络中的节点无需自主执行以下操作：

 - 选择或选择根节点（请参阅[构建网络](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/mesh.html#mesh-building-a-network)中的自动根节点选择）
 - 选择首选父节点（请参阅[构建网络](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/mesh.html#mesh-building-a-network)中的父节点选择）
 - 检测到断开连接时自动重新连接（请参阅[管理网络](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/mesh.html#mesh-managing-a-network)中的中间父节点故障）

启用自组织网络后，ESP-MESH 栈将在内部调用 Wi-Fi 驱动程序 API。因此，应用层不应对 Wi-Fi 驱动程序 API 进行任何调用，同时启用自组织网络，否则可能会干扰 ESP-MESH。

### 切换自组织网络

应用程序在运行时通过调用 `esp_mesh_set_self_organized()` 函数可以启用或禁用自组织网络。该函数具有以下两个参数：

 - `bool enable` 指定是启用还是禁用自组织网络。
 - `bool select_parent` 指定在启用自组织网络时是否应选择新的父节点。根据节点类型和节点的当前状态，选择新父级具有不同的效果。禁用自组织网络时，此参数未使用。

### 禁用自组织网络

以下代码段演示了如何禁用自组织网络。

```
//Disable self organized networking
esp_mesh_set_self_organized(false, false);
```

当禁用自组织网络时，ESP-MESH 将尝试维持节点当前的 Wi-Fi 状态。

 - 如果节点先前已连接到其他节点，则它将保持连接状态。
 - 如果节点先前已断开连接并且正在扫描父节点或路由器，则它将停止扫描。
 - 如果节点先前尝试重新连接到父节点或路由器，它将停止重新连接。

### 启用自组织网络

在启用自组织网络时，ESP-MESH 将尝试维持节点当前的 Wi-Fi 状态。但是，根据节点类型以及是否选择了新父节点，节点的 Wi-Fi 状态可能会发生变化。下表显示了启用自组织网络的效果。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190302173636709.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI3MTE0Mzk3,size_16,color_FFFFFF,t_70)
以下代码演示了如何启用自组织网络。

```
//Enable self organized networking and select a new parent
esp_mesh_set_self_organized(true, true);

...

//Enable self organized networking and manually reconnect
esp_mesh_set_self_organized(true, false);
esp_mesh_connect();
```

### 调用 Wi-Fi 驱动程序 API

应用程序可能希望在使用 ESP-MESH 时直接调用 Wi-Fi 驱动程序 API 的情况。例如，应用程序可能想要手动扫描相邻的 AP。但是，**在应用程序调用任何 Wi-Fi 驱动程序 API 之前，必须禁用自组织网络。** 这将阻止 ESP-MESH 栈尝试调用任何 Wi-Fi 驱动程序 API ，ESP-MESH 栈的这些调用可能干扰应用程序的调用。

因此，**应该在调用 `esp_mesh_set_self_organized()` 进行禁用和启用自组织网络之间放置对 Wi-Fi 驱动程序 API 的调用。** 以下代码段演示了应用程序在使用 ESP-MESH 时如何安全地调用 `esp_wifi_scan_start()`。

```
//Disable self organized networking
esp_mesh_set_self_organized(0, 0);

//Stop any scans already in progress
esp_wifi_scan_stop();
//Manually start scan. Will automatically stop when run to completion
esp_wifi_scan_start();

//Process scan results

...

//Re-enable self organized networking if still connected
esp_mesh_set_self_organized(1, 0);

...

//Re-enable self organized networking if non-root and disconnected
esp_mesh_set_self_organized(1, 1);

...

//Re-enable self organized networking if root and disconnected
esp_mesh_set_self_organized(1, 0);  //Don't select new parent
esp_mesh_connect();                 //Manually reconnect to router
```

## 应用实例

ESP-IDF 包含这些 ESP-MESH 示例项目：

[内部通信示例](https://github.com/espressif/esp-idf/tree/ebdcbe8c6/examples/mesh/internal_communication)演示了如何设置 ESP-MESH 网络并让根节点向网络中的每个节点发送数据包。

[手动网络示例](https://github.com/espressif/esp-idf/tree/ebdcbe8c6/examples/mesh/manual_networking)演示了如何在没有自组织功能的情况下使用 ESP-MESH。此示例显示如何对节点进行编程以手动扫描潜在父节点列表，并根据自定义条件选择父节点。

## API 参考

### 头文件

 - esp32/include/esp_mesh.h

### 相关函数

 - esp_err_tesp_mesh_init(void)
 - esp_err_tesp_mesh_deinit(void)
 - esp_err_tesp_mesh_start(void)
 - esp_err_tesp_mesh_stop(void)
 - esp_err_tesp_mesh_send(constmesh_addr_t *to, constmesh_data_t *data, int flag, constmesh_opt_topt[], int opt_count)
 - esp_err_tesp_mesh_recv(mesh_addr_t *from, mesh_data_t *data, int timeout_ms, int *flag, mesh_opt_topt[], int opt_count)
 - esp_err_tesp_mesh_recv_toDS(mesh_addr_t *from, mesh_addr_t *to, mesh_data_t *data, int timeout_ms, int *flag, mesh_opt_topt[], int opt_count)
 - esp_err_tesp_mesh_set_config(constmesh_cfg_t *config)
 - esp_err_tesp_mesh_get_config(mesh_cfg_t *config)
 - esp_err_tesp_mesh_set_router(constmesh_router_t *router)
 - esp_err_tesp_mesh_get_router(mesh_router_t *router)
 - esp_err_tesp_mesh_set_id(constmesh_addr_t *id)
 - esp_err_tesp_mesh_get_id(mesh_addr_t *id)
 - esp_err_tesp_mesh_set_type(mesh_type_ttype)
 - mesh_type_tesp_mesh_get_type(void)
 - esp_err_tesp_mesh_set_max_layer(int max_layer)
 - int esp_mesh_get_max_layer(void)
 - esp_err_tesp_mesh_set_ap_password(const uint8_t *pwd, int len)
 - esp_err_tesp_mesh_set_ap_authmode(wifi_auth_mode_tauthmode)
 - wifi_auth_mode_tesp_mesh_get_ap_authmode(void)
 - esp_err_tesp_mesh_set_ap_connections(int connections)
 - int esp_mesh_get_ap_connections(void)
 - int esp_mesh_get_layer(void)
 - esp_err_tesp_mesh_get_parent_bssid(mesh_addr_t *bssid)
 - bool esp_mesh_is_root(void)
 - esp_err_tesp_mesh_set_self_organized(bool enable, bool select_parent)
 - bool esp_mesh_get_self_organized(void)
 - esp_err_tesp_mesh_waive_root(constmesh_vote_t *vote, int reason)
 - esp_err_tesp_mesh_set_vote_percentage(float percentage)
 - float esp_mesh_get_vote_percentage(void)
 - esp_err_tesp_mesh_set_ap_assoc_expire(int seconds)
 - int esp_mesh_get_ap_assoc_expire(void)
 - int esp_mesh_get_total_node_num(void)
 - int esp_mesh_get_routing_table_size(void)
 - esp_err_tesp_mesh_get_routing_table(mesh_addr_t *mac, int len, int *size)
 - esp_err_tesp_mesh_post_toDS_state(bool reachable)
 - esp_err_tesp_mesh_get_tx_pending(mesh_tx_pending_t *pending)
 - esp_err_tesp_mesh_get_rx_pending(mesh_rx_pending_t *pending)
 - int esp_mesh_available_txupQ_num(constmesh_addr_t *addr, uint32_t *xseqno_in)
 - esp_err_tesp_mesh_set_xon_qsize(int qsize)
 - int esp_mesh_get_xon_qsize(void)
 - esp_err_tesp_mesh_allow_root_conflicts(bool allowed)
 - bool esp_mesh_is_root_conflicts_allowed(void)
 - esp_err_tesp_mesh_set_group_id(constmesh_addr_t *addr, int num)
 - esp_err_tesp_mesh_delete_group_id(constmesh_addr_t *addr, int num)
 - int esp_mesh_get_group_num(void)
 - esp_err_tesp_mesh_get_group_list(mesh_addr_t *addr, int num)
 - bool esp_mesh_is_my_group(constmesh_addr_t *addr)
 - esp_err_tesp_mesh_set_capacity_num(int num)
 - int esp_mesh_get_capacity_num(void)
 - esp_err_tesp_mesh_set_ie_crypto_funcs(const mesh_crypto_funcs_t *crypto_funcs)
 - esp_err_tesp_mesh_set_ie_crypto_key(const char *key, int len)
 - esp_err_tesp_mesh_get_ie_crypto_key(char *key, int len)
 - esp_err_tesp_mesh_set_root_healing_delay(int delay_ms)
 - int esp_mesh_get_root_healing_delay(void)
 - esp_err_tesp_mesh_set_event_cb(constmesh_event_cb_tevent_cb)
 - esp_err_tesp_mesh_fix_root(bool enable)
 - bool esp_mesh_is_root_fixed(void)
 - esp_err_tesp_mesh_set_parent(constwifi_config_t *parent, constmesh_addr_t *parent_mesh_id, mesh_type_tmy_type, int my_layer)
 - esp_err_tesp_mesh_scan_get_ap_ie_len(int *len)
 - esp_err_tesp_mesh_scan_get_ap_record(wifi_ap_record_t *ap_record, void *buffer)
 - esp_err_tesp_mesh_flush_upstream_packets(void)
 - esp_err_tesp_mesh_get_subnet_nodes_num(constmesh_addr_t *child_mac, int *nodes_num)
 - esp_err_tesp_mesh_get_subnet_nodes_list(constmesh_addr_t *child_mac, mesh_addr_t *nodes, int nodes_num)
 - esp_err_tesp_mesh_disconnect(void)
 - esp_err_tesp_mesh_connect(void)
 - esp_err_tesp_mesh_flush_scan_result(void)
 - esp_err_tesp_mesh_switch_channel(const uint8_t *new_bssid, int csa_newchan, int csa_count)
 - esp_err_tesp_mesh_get_router_bssid(uint8_t *router_bssid)

## 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/network/esp_mesh.html)
