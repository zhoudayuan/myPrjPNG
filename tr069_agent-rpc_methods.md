@startuml
|#AliceBlue|方法|
start
fork
    split

    :uloop_timeout_set;
    :实例: inform_timer\n{ .cb = cwmp_do_inform };

    split again
    :实例: inform_timer_retry\n{ .cb = cwmp_do_inform };

    end split

    :cwmp_do_inform;
    :cwmp_inform;
    :cwmp_handle_messages;
    partition 根据不同RPC方法调用不同的函数进行处理 {
        :xml_handle_message;
        : 遍历rpc_methods;
        :if (method->handler(b, tree_in, tree_out)) goto error;
    }
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
|#azure|消息-end_msg_to_oam|
    :定时器实例:inform_timer\n回调:cb = cwmp_do_inform;
    note
        初始化:
        cwmp_add_inform_timer
        uloop_timeout_set(&inform_timer, 10)
        10ms进一次中断
    endnote
    :cwmp_inform;
    :cwmp_handle_messages;
    :xml_handle_message;
    :rpc_methods;
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
    :xml_handle_set_parameter_values;
    note
        SetParameterValues
    endnote

    partition handle_set_parameter_values {
        :tr069_set_parameter_values_req;
        :send_msg_to_oam;
    }



fork again
|#thistle|消息-tr069_set_parameter_values_req|
    :AAAAAA;

end fork
stop


@enduml
