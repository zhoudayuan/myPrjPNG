@startuml
|#AliceBlue|方法|
start
fork
    :main;
    :CSPL_InitEx 初始化日志\n替换 CSPL_Init 和 CSPL_InitLog 和 CSPL_Open 函数;
    :初始化 cwmp
    初始化链表
    INIT_LIST_HEAD(&cwmp->events)
    INIT_LIST_HEAD(&cwmp->notifications)
    INIT_LIST_HEAD(&cwmp->downloads)
    INIT_LIST_HEAD(&cwmp->uploads)
    INIT_LIST_HEAD(&cwmp->scheduled_informs);
    note
        struct cwmp_internal {
            struct list_head events;            // 事件链表
            struct list_head notifications;     // 通知链表
            struct list_head downloads;         // 下载链表
            struct list_head uploads;           // 上传链表
            struct list_head scheduled_informs; // 计划通知链表
            struct deviceid deviceid;           // 设备信息
            int retry_count;
            int download_count;
            int upload_count;
            int end_session;
            int method_id;
            bool get_rpc_methods;
            bool hold_requests;
            int netlink_sock[2];
        };
    endnote


fork again
|#pink|方法-2|
    :callback:cwmp_do_inform@inform_timer;
    :cwmp_do_inform;
    :cwmp_inform;

    :cwmp_add_event
        add event '2 PERIODIC';
    :event_code_array;

fork again
|#khaki|线程-cspl_oam_thread|
:功能: 开启CSPL线程，
        用于和OAM之间进行首发消息\n入参: oam的通信IP
实体: entityRcvFromOAM;
:接收消息: process_rvcmsg_from_oam;


fork again
|#cyan|回调-cwmp_netlink_interface|
    :注册: netlink_event\n类型: uloop;
    :cwmp_netlink_interface;
    note
        static struct uloop_fd netlink_event
        = { .cb = netlink_new_msg };
    endnote
    :cwmp_netlink_interface;

fork again
|#azure|流程:【tr069_agent连接网管】\n关键函数:cwmp_inform\n网管消息:SetParameterValues\n 网管 ==> tr069_agent|
    :定时器实例:inform_timer\n回调:cb = cwmp_do_inform;
    note
        初始化:
        cwmp_add_inform_timer
        uloop_timeout_set(&inform_timer, 10)
        10ms进一次中断
    endnote

    :cwmp_inform;


    partition "主动连接" {

        :http_client_init;
        note
            0-鉴权
                config->acs->ssl_cert
                config->acs->ssl_cacert
            1-连接ACS时 认证用户名和密码信息
            2-用户名密码从config里来
        endnote
        :callback:http_get_response;




    }

    partition "构造inform消息，\n发送给acs，同时接收acs的响应消息，解析InformResponse消息\n入口:rpc_inform" {
        :rpc_inform;
        :xml_prepare_inform_message;
        note
            装载 inform-报文：CWMP_INFORM_MESSAGE 获取指针
        endnote

    }

    :cwmp_handle_messages\ninform-消息之后, RPC请求命令，调用函数处理ACS发送的消息 ;

    partition xml_handle_message\nrpc-找到操作方法 {
        :xml_handle_message;
        :根据返回值 xml_mxml_find_node_by_env_type
        遍历 rpc_methods;
        note
            const struct rpc_method rpc_methods[] = {
                { "GetRPCMethods",          xml_handle_get_rpc_methods },
                { "SetParameterValues",     xml_handle_set_parameter_values },
                { "GetParameterValues",     xml_handle_get_parameter_values },
                { "GetParameterNames",      xml_handle_get_parameter_names },
                { "GetParameterAttributes", xml_handle_get_parameter_attributes },
                { "SetParameterAttributes", xml_handle_set_parameter_attributes },
                { "AddObject",              xml_handle_AddObject },
                { "DeleteObject",           xml_handle_DeleteObject },
                { "Download",               xml_handle_download },
                { "Upload",                 xml_handle_upload },
                { "Reboot",                 xml_handle_reboot },
                { "FactoryReset",           xml_handle_factory_reset },
                { "ScheduleInform",         xml_handle_schedule_inform },
            };
        endnote
    }

    :xml_handle_set_parameter_values;
    note
        SetParameterValues
    endnote

    partition handle_set_parameter_values {
        :tr069_set_parameter_values_req;
        :send_msg_to_oam;
    }
fork again
|#wheat|rpc_methods-SetParameterValues|
    :xml_handle_set_parameter_values;

fork again
|#thistle|消息-AAAAAAAAAAAAAAAAAAAAAAAAA|
    :AAAAAAAAAAAAAAAAAAAAAAAAA;

end fork
stop


@enduml
