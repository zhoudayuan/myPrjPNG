@startuml

|#siliver|初始化-json|
start
    fork

        :oam_pm_early_init;
        :oam_pm_realtime_template_parse;
        note
            参数: 全局: OAM_PM_TEMPLATE_LIST *g_temp = NULL;
        endnote
        :oam_pm_parse_performance_template;
        partition 读取文件，获取pRootJson {
            :oam_pm_parse_json_file;
            note
                返回：cJSON *pRootJson = NULL;
            endnote


            :oam_pm_read_json_file;
            note
                读取json文件内容：根据文件名读取，将内存转为char * 存储
            endnote

            :cJSON_Parse;
            note
                cJSON *pJson: 将char *文件内容格式化cJSON格式存储
            endnote
        }




        :字段匹配: cJSON_GetObjectItem(pRootJson, "neVersion");




        note
            返回值是cJSON *类型;
            ====
            /* The cJSON structure: */
            typedef struct cJSON {
                struct cJSON *next,*prev;
                struct cJSON *child;
                int     type;
                char   *valuestring;
                int     valueint;
                double  valuedouble;
                char   *string;
            } cJSON;
        endnote

        :字段匹配: cJSON_GetObjectItem(pRootJson, "neTypeId");
        :字段匹配: cJSON_GetObjectItem(pRootJson, "neTypeName");
        :字段匹配: cJSON_GetObjectItem(pRootJson, "measureObjTypeList");



        partition 对公共部分赋值，把json中的数据存放在q【类型:OAM_PM_TEMPLATE_LIST】中 {
            :公共部分: neVersion,neTypeId,neTypeName;
            note
                q->objectType = NULL;
                q->next = NULL;
                *pTemp = q;
                pTemplate = q;
            endnote
        }



        partition 解析数组measureObjTypeList中数据{
            :oam_pm_parse_measure_obj_type_list ;
            :cJSON_GetArraySize----获取measureObjTypeList数组大小;
        }



        while (遍历measureObjTypeList对应的数组)
            :取出数组中元素-根据i(索引：接口内部遍历查询)-cJSON_GetArrayItem;
            note
                返回值: cJSON *pSubJson = NULL;
            endnote

            :cJSON *cJSON_GetObjectItem(cJSON *object, const char *string);
            note
                遍历，接口对比，找到该json：
                js_measureObjTypeId = cJSON_GetObjectItem(pSubJson, "measureObjTypeId");
                js_name = cJSON_GetObjectItem(pSubJson, "name");
                js_commAttributes = cJSON_GetObjectItem(pSubJson, "commAttributes");
                js_commAttributeVals = cJSON_GetObjectItem(pSubJson, "commAttributeVals");
                js_measureObjList = cJSON_GetObjectItem(pSubJson, "measureObj");
            endnote


        :oam_pm_parse_common_attribute-解析数组commAttribute中数据;
        :oam_pm_parse_measure_obj_list-解析数组measureObjList中数据;

        endwhile


    fork again
    |#lightcoral|输出:处理业务实时数据上报|
        :oam_pm_intf_process_timer;
        :case OAM_PM_SECOND_TIMEOUT_TYPE;
        :oam_pm_deal_report_mod_data;
        :oam_pm_report_data_to_nms;
            :CSPL_GetSysTime\n获取当前系统时间;
        :oam_pm_get_ne_version\n获取网元版本号;
        :oam_pm_connect_to_redis\n与redis上指定数据库建立连接;
        :cJSON_CreateObject\n与redis上指定数据库建立连接;


        partition 获取counter采样类型并写入json {
            :oam_pm_get_counter_mode;
            note
                参数-2: moId: measureObjId : task->moId;
                获取: dataUpPeriodMod

            endnote
            :获取链表中的dataUpPeriodMod返回;
        }




        :cJSON_AddStringToObject;
        :oam_pm_get_mo_type_Id \n获取moTypeId来判断公共属性;
        note
            实际返回: json中的measureObjTypeId对应数值
                       在measureObjTypeList中
        endnote


        :oam_snmp_varlist_add_variable;
        partition 通过snmp向网管发送数据 {
            :oam_snmp_send_trap;
            note
                : 参数-2:   SNMP_KPI
            endnote

        }

    fork again
        |#darkseagreen|OAM_PM_REALTIME_NMS_TASK的来源：MML命令|
        :oam_pm_intf_process_msg;
        :case:MML_OAM_PM_RT_TASK_ADD_REQ;

        :oam_pm_realtime_deal_mml_req \nPM master处理mml任务请求;
        note
            case MML_OAM_PM_RT_TASK_ADD_REQ:
            case MML_OAM_PM_RT_TASK_DEL_REQ:
            case MML_OAM_PM_RT_TASK_QRY_REQ:
            case MML_OAM_PM_RT_TASK_SYN_REQ:
            case MML_OAM_PM_RT_TASK_RST_REQ:
        endnote



        partition 处理网管下发启动采集 {
            :oam_pm_realtime_deal_add_req \n处理网管下发启动采集;
            :oam_pm_realtime_get_mo_cnt_num \n解析CSPL消息，获取moSite和counter数目;
            note
                从mml add消息内容解析mo和cnt数目
            endnote

            :申请内存空间保存新task;
            note
                根据解析出的moSiteNum 来分配内存：OAM_PM_REALTIME_NMS_TASK
            endnote

            :oam_pm_realtime_mml_add_msg \nTLV格式解析mml add消息内容;
            note
                解析CSPL
                获取 OAM_PM_REALTIME_NMS_TASK *task
                获取 OAM_PM_REALTIME_NMS_CNT *cnt
            endnote


            :oam_pm_realtime_is_task_full \n判断实时task列表是否已满;
            note
            检查 实时task列表
            i.e.  g_stRealtimeTask[OAM_PM_TASK_MAX];
            原型: OAM_PM_REALTIME_TASK_LIST

            endnote

            :oam_pm_realtime_is_task_exist \n判断实时task是否已存在;
            :oam_pm_realtime_add_to_redis \n实时task挂载在task列表上;
            :oam_pm_realtime_mount_to_list;
            :oam_pm_realtime_send_rsp_to_mml;
        }


    endfork
stop
@enduml

