# BearPi-HM_Nano开发板智能台灯案例开发
本示例将演示如何在BearPi-HM_Nano开发板上使用MQTT协议连接华为IoT平台，使用E53_SC1 智能台灯扩展板与 BearPi-HM_Nano 开发板实现智能台灯的案例，设备安装如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/E53_SC1安装.png "E53_SC1安装")

## 软件设计

### 连接平台
在连接平台前需要设置获取CONFIG_APP_DEVICEID、CONFIG_APP_DEVICEPWD、CONFIG_APP_SERVERIP、CONFIG_APP_SERVERPORT，通过oc_mqtt_profile_connect()函数连接平台。
```c
    WifiConnect(CONFIG_WIFI_SSID, CONFIG_WIFI_PWD);
    dtls_al_init();
    mqtt_al_init();
    oc_mqtt_init();
    
    g_app_cb.app_msg = queue_create("queue_rcvmsg",10,1);
    if(NULL ==  g_app_cb.app_msg){
        printf("Create receive msg queue failed");
        
    }
    oc_mqtt_profile_connect_t  connect_para;
    (void) memset( &connect_para, 0, sizeof(connect_para));

    connect_para.boostrap =      0;
    connect_para.device_id =     CONFIG_APP_DEVICEID;
    connect_para.device_passwd = CONFIG_APP_DEVICEPWD;
    connect_para.server_addr =   CONFIG_APP_SERVERIP;
    connect_para.server_port =   CONFIG_APP_SERVERPORT;
    connect_para.life_time =     CONFIG_APP_LIFETIME;
    connect_para.rcvfunc =       msg_rcv_callback;
    connect_para.security.type = EN_DTLS_AL_SECURITY_TYPE_NONE;
    //连接平台
    ret = oc_mqtt_profile_connect(&connect_para);
    if((ret == (int)en_oc_mqtt_err_ok)){
        g_app_cb.connected = 1;
        printf("oc_mqtt_profile_connect succed!\r\n");
    }
    else
    {
        printf("oc_mqtt_profile_connect faild!\r\n");
    }
```

### 推送数据

当需要上传数据时，需要先拼装数据，让后通过oc_mqtt_profile_propertyreport上报数据。代码示例如下： 

```c
static void deal_report_msg(report_t *report)
{
    oc_mqtt_profile_service_t service;
    oc_mqtt_profile_kv_t luminance;
    oc_mqtt_profile_kv_t led;

    if (g_app_cb.connected != 1) {
        return;
    }

    service.event_time = NULL;
    service.service_id = "Light";
    service.service_property = &luminance;
    service.nxt = NULL;
    
    luminance.key = "Luminance";
    luminance.value = &report->lum;
    luminance.type = EN_OC_MQTT_PROFILE_VALUE_INT;
    luminance.nxt = &led;

    led.key = "LightStatus";
    led.value = g_app_cb.led ? "ON" : "OFF";
    led.type = EN_OC_MQTT_PROFILE_VALUE_STRING;
    led.nxt = NULL;
    //发送数据
    oc_mqtt_profile_propertyreport(NULL,&service);
    return;
}
```




### 命令接收

华为IoT平台支持下发命令，命令是用户自定义的。接收到命令后会将命令数据发送到队列中，CloudMainTaskEntry函数中读取队列数据并调用deal_cmd_msg函数进行处理，代码示例如下： 

```c

static int msg_rcv_callback(oc_mqtt_profile_msgrcv_t *msg)
{
    int    ret = 0;
    char  *buf;
    int    buf_len;
    app_msg_t *app_msg;

    if ((NULL == msg)|| (msg->request_id == NULL) || (msg->type != EN_OC_MQTT_PROFILE_MSG_TYPE_DOWN_COMMANDS)) {
        return ret;
    }

    buf_len = sizeof(app_msg_t) + strlen(msg->request_id) + 1 + msg->msg_len + 1;
    buf = malloc(buf_len);
    if (NULL == buf) {
        return ret;
    }
    app_msg = (app_msg_t *)buf;
    buf += sizeof(app_msg_t);

    app_msg->msg_type = en_msg_cmd;
    app_msg->msg.cmd.request_id = buf;
    buf_len = strlen(msg->request_id);
    buf += buf_len + 1;
    memcpy(app_msg->msg.cmd.request_id, msg->request_id, buf_len);
    app_msg->msg.cmd.request_id[buf_len] = '\0';

    buf_len = msg->msg_len;
    app_msg->msg.cmd.payload = buf;
    memcpy(app_msg->msg.cmd.payload, msg->msg, buf_len);
    app_msg->msg.cmd.payload[buf_len] = '\0';

    ret = queue_push(g_app_cb.app_msg,app_msg,10);
    if (ret != 0) {
        free(app_msg);
    }

    return ret;
}

///< COMMAND DEAL
#include <cJSON.h>
static void deal_cmd_msg(cmd_t *cmd)
{
    cJSON *obj_root;
    cJSON *obj_cmdname;
    cJSON *obj_paras;
    cJSON *obj_para;

    int cmdret = 1;
    oc_mqtt_profile_cmdresp_t cmdresp;
    obj_root = cJSON_Parse(cmd->payload);
    if (NULL == obj_root) {
        goto EXIT_JSONPARSE;
    }

    obj_cmdname = cJSON_GetObjectItem(obj_root, "command_name");
    if (NULL == obj_cmdname) {
        goto EXIT_CMDOBJ;
    }
    if (0 == strcmp(cJSON_GetStringValue(obj_cmdname), "Light_Control_Led")) {
        obj_paras = cJSON_GetObjectItem(obj_root, "paras");
        if (NULL == obj_paras) {
            goto EXIT_OBJPARAS;
        }
        obj_para = cJSON_GetObjectItem(obj_paras, "Led");
        if (NULL == obj_para) {
            goto EXIT_OBJPARA;
        }
        ///< operate the LED here
        if (0 == strcmp(cJSON_GetStringValue(obj_para), "ON")) {
            g_app_cb.led = 1;
            Light_StatusSet(ON);
            printf("Led On!\r\n");
        } else {
            g_app_cb.led = 0;
            Light_StatusSet(OFF);
            printf("Led Off!\r\n");
        }
        cmdret = 0;
    }
    

EXIT_OBJPARA:
EXIT_OBJPARAS:
EXIT_CMDOBJ:
    cJSON_Delete(obj_root);
EXIT_JSONPARSE:
    ///< do the response
    cmdresp.paras = NULL;
    cmdresp.request_id = cmd->request_id;
    cmdresp.ret_code = cmdret;
    cmdresp.ret_name = NULL;
    (void)oc_mqtt_profile_cmdresp(NULL, &cmdresp);
    return;
}
```


## 编译调试


### 登录

设备接入华为云平台之前，需要在平台注册用户账号，华为云地址：<https://www.huaweicloud.com/>

在华为云首页单击产品，找到IoT物联网，单击设备接入IoTDA 并单击立即使用，如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/登录平台01.png "登录平台")

![](../../docs/figures/D8_iot_cloud_oc_light/登录平台02.png "登录平台")

### 创建产品

在设备接入页面可看到总览界面，展示了华为云平台接入的协议与域名信息，根据需要选取MQTT通讯必要的信息备用，如下图所示。

接入协议（端口号）：MQTT 1883

域名：iot-mqtts.cn-north-4.myhuaweicloud.com

![](../../docs/figures/D8_iot_cloud_oc_light/查看平台信息.png "查看平台信息")

选中侧边栏产品页，单击右上角“创建产品”，在页面中选中所属资源空间，并且按要求填写产品名称，选中MQTT协议，数据格式为JSON，并填写厂商名称，在下方模型定义栏中选择所属行业以及添加设备类型，并单击右下角“确定”，如下图所示。：

![](../../docs/figures/D8_iot_cloud_oc_light/创建产品01.png "创建产品")



创建完成后，在产品页会自动生成刚刚创建的产品，单击“查看”可查看创建的具体信息，如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/查看产品.png "查看产品")


单击产品详情页的自定义模型，在弹出页面中新增服务，如下图所示。

服务ID：`Light`(必须一致)

服务类型：`Senser`(可自定义)
![](../../docs/figures/D8_iot_cloud_oc_light/创建产品02.png "创建产品")

在“Light”的下拉菜单下点击“添加属性”填写相关信息“Luminance”，“LightStatus”，如下图所示。


![](../../docs/figures/D8_iot_cloud_oc_light/创建产品03.png "创建产品")

![](../../docs/figures/D8_iot_cloud_oc_light/创建产品04.png "创建产品")

在“Light”的下拉菜单下点击“添加命令”填写相关信息，如下图所示。

命令名称：`Light_Control_Led`

参数名称：`Led`

数据类型：`string`

长度：`3`

枚举值：`ON,OFF`

![](../../docs/figures/D8_iot_cloud_oc_light/创建产品05.png "创建产品")


#### 注册设备

在侧边栏中单击“设备”，进入设备页面，单击右上角“注册设备”，勾选对应所属资源空间并选中刚刚创建的产品，注意设备认证类型选择“秘钥”，按要求填写秘钥，如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/注册设备01.png "注册设备")

记录下设备ID和设备密钥，如下图所示。
![](../../docs/figures/D8_iot_cloud_oc_light/注册设备02.png "注册设备")

注册完成后，在设备页面单击“所有设备”，即可看到新建的设备，同时设备处于未激活状态，如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/注册设备03.png "注册设备")


### 修改代码中设备信息
修改`iot_cloud_oc_sample.c`中第31行附近的wifi的ssid和pwd，以及设备的DEVICEID和DEVICEPWD（这两个参数是在平台注册设备时产生的），如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/修改设备信息.png "修改设备信息")

### 修改 BUILD.gn 文件

修改 `device\bearpi\bearpi_hm_nano\app`路径下 BUILD.gn 文件，指定 `cloud_oc_light` 参与编译。

```r
#"D7_iot_cloud_oc_smoke:cloud_oc_smoke",
"D8_iot_cloud_oc_light:cloud_oc_light",
#"D9_iot_cloud_oc_manhole_cover:cloud_oc_manhole_cover",
#"D10_iot_cloud_oc_infrared:cloud_oc_infrared",
#"D11_iot_cloud_oc_agriculture:cloud_oc_agriculture",
#"D12_iot_cloud_oc_gps:cloud_oc_gps",
```
### 测试


示例代码编译烧录代码后，按下开发板的RESET按键，平台上的设备显示为在线状态，如下图所示。

![](../../docs/figures/D8_iot_cloud_oc_light/设备在线.png "设备在线")
    
点击设备右侧的“查看”，进入设备详情页面，可看到上报的数据。



在华为云平台设备详情页，单击“命令”，选择同步命令下发，选中创建的命令属性，单击“确定”，即可发送下发命令控制设备。
