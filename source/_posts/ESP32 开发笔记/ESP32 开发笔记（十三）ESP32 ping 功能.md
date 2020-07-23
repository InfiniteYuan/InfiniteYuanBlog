---
title: ESP32 开发笔记（十三）ESP32 ping 功能
date: 2019-11-22 01:23:18
categories:
- ESP32 开发笔记
tags:
- ESP32
---

# ping


```c
esp_err_t pingResults(ping_target_id_t msgType, esp_ping_found * pf)
{
    printf("AvgTime:%.1fmS Sent:%d Rec:%d min(mS):%d max(mS):%d Resp(mS):%d Timeouts:%d Total Time:%d\n", (float)pf->total_time/pf->recv_count, pf->send_count, pf->recv_count,  pf->min_time, pf->max_time ,pf->resp_time, pf->timeout_count, pf->total_time);
    return ESP_OK;
}
void Start_Ping_ESP32(void){
 
    uint32_t ip_pinG = ipaddr_addr("163.177.151.109");//百度IP地址
    uint32_t ping_count = 2000;//ping的次数
    uint32_t ping_timeout = 1000; //超时时间
    uint32_t ping_delay = 2000; //ping 的间隔
    ping_deinit();
    esp_ping_set_target(PING_TARGET_IP_ADDRESS_COUNT, &ping_count, sizeof(uint32_t));
    esp_ping_set_target(PING_TARGET_RCV_TIMEO, &ping_timeout, sizeof(uint32_t));
    esp_ping_set_target(PING_TARGET_DELAY_TIME, &ping_delay, sizeof(uint32_t));
    esp_ping_set_target(PING_TARGET_IP_ADDRESS, &ip_pinG, sizeof(uint32_t));
    esp_ping_set_target(PING_TARGET_RES_FN, &pingResults, sizeof(pingResults));
    
    uint8_t res = 0;
    res = ping_init();
    if(res == 0){
        printf("PING!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!\n");
    }else{
        printf("error:%d\n",res);
    }
}
```
