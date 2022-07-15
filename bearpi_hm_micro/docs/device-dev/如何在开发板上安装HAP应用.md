# 如何在开发板上安装HAP应用

以下以安装一个控制灯开关的应用为例讲解如何在开发板上安装HAP应用。

## 一、准备工作

- 准备一张SD卡（需要格式化为FAT32），以及一个读卡器。

## 二、安装HAP应用

1. 将device/board/bearpi/bearpi_hm_micro/tools/hap_tools/hap_example路径下LED_1.0.0.hap拷贝到SD卡中；

    将out\bearpi_hm_micro\bearpi_hm_micro\dev_tools\bin路径下的bm拷贝到SD卡中。

2. 将SD卡插入到开发板中，并按开发板的RESET按键重启开发板。

3. 输入以下命令，进入SD卡目录
	```
    cd /sdcard
    ```
4. 输入以下命令，打开调试模式
    ```
    ./bm set -s disable
    ./bm set -d enable
    ```
    
5.	安装应用
    ```
    ./bm install -p LED_1.0.0.hap
    ```

注: LED_1.0.0.hap为安装包名称，安装其他应用需要修改为对应的安装包名称。
