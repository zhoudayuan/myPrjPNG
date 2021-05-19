@startuml
title TR069_GET_PARAM-流程

participant  tr069_agent

box "entitytr069ParseThread" #LightBlue
participant  oam_adp_tr069.c
participant  l3_adapter.c
end box



box "adpLayerEntity" #LightPink
participant  oam_adp_layer.c
participant  cfg_mom.c
participant  forward_channel_thread.c
participant  mapping.c
participant  oam_crm_main.c
participant  oam_utils.c
participant  mapping_xml.c
end box
participant  cspl

tr069_agent -[#Green]> oam_adp_tr069.c : TR069_GET_PARAM(api_id<0x1537>)
note left
CSPL-携带消息:
typedef struct{
	U32 data_path_num;
	char data_path[DATA_PATH_MAX][256];
}tr069_get_t;
end note





oam_adp_tr069.c -> oam_adp_tr069.c: oam_adp_tr069_agent_process_msg
activate oam_adp_tr069.c
alt case:TR069_GET_PARAM
    group 消息转换\n将tr069_agent消息转换为适配层(tlv)消息\n函数实现:tr069_process_get_msg
        oam_adp_tr069.c->oam_adp_tr069.c
        activate   oam_adp_tr069.c
        oam_adp_tr069.c->l3_adapter.c : getDataPathALL
        activate   l3_adapter.c
        group 转换处理\n函数实现:getDataPathALL\n参数-1:<输入>datapath\n参数-2:<输出>datapathallnum\n参数-3:<输出>datapathall
            l3_adapter.c -> l3_adapter.c : 消息转换: MO-->MO.Index.LocIpAddrList\n查表Mo_name_Func\n根据MO执行表中处理，返回转换的消息\nMoFunc(datapath, datapathallnum, datapathall)
            activate l3_adapter.c
            deactivate l3_adapter.c

        end
        deactivate l3_adapter.c
        oam_adp_tr069.c->oam_adp_tr069.c: tr069_send_one_get_msg
        activate oam_adp_tr069.c
        oam_adp_tr069.c -> oam_adp_tr069.c: send_msg_to_adp_cfg
        activate oam_adp_tr069.c
        deactivate oam_adp_tr069.c
        deactivate oam_adp_tr069.c

        deactivate oam_adp_tr069.c
    end
end

oam_adp_tr069.c-[#Green]>oam_adp_layer.c :apiID:0x1fff(Msg: V_LTEOAM__OPERATE_TYPE_E__GET_VALUE)
    note left
        CSPL-携带消息:
        SRC:<248>
        DST:<251>
        API:<0x1fff>
        msg:V_LTEOAM__OPERATE_TYPE_E__GET_VALUE
    end note
deactivate oam_adp_tr069.c



oam_adp_layer.c -> oam_adp_layer.c : oam_adp_layer_intf_process_msg
activate oam_adp_layer.c
oam_adp_layer.c -> cfg_mom.c : oam_adp_tr069_req_process
group 解析tr069转换后的消息\n函数实现: oam_adp_tr069_req_process
    cfg_mom.c->mapping.c : oam_create_mo_oper
    activate mapping.c
    deactivate mapping.c
    mapping.c -> mapping_xml.c: find_map_obj
    activate mapping_xml.c
    deactivate mapping_xml.c


    alt V_LTEOAM__OPERATE_TYPE_E__GET_VALUE
        cfg_mom.c -> mapping.c : oam_merge_mo_oper
        cfg_mom.c -> cfg_mom.c : process_some_request
        activate cfg_mom.c
        group 函数实现: process_some_request
            cfg_mom.c -> cfg_mom.c : ret=process_request
            group 函数实现:process_request
                activate cfg_mom.c
                    cfg_mom.c->cfg_mom.c: send_msg

                    activate cfg_mom.c
                        cfg_mom.c -> cfg_mom.c: get_msg_seq
                        cfg_mom.c -> cfg_mom.c: getMoMetaById
                        cfg_mom.c -> cfg_mom.c: getActMetaByCode
                        cfg_mom.c -> forward_channel_thread.c: 构造list并入队,业务调用此接口发送cspl\noam_adp_frm_passthrough_msg
                        forward_channel_thread.c->forward_channel_thread.c : 进入发送队列\nforward_channel_send_query_list
                        activate forward_channel_thread.c
                        deactivate forward_channel_thread.c
                        alt is_rollback=false
                        cfg_mom.c->cfg_mom.c: start_timer
                        activate cfg_mom.c
                            cfg_mom.c->oam_utils.c: oam_start_new_timer
                            activate oam_utils.c
                            alt 0==expiry_thread_id
                            oam_utils.c-> cspl : qvTimerStart
                            else 0!=expiry_thread_id
                            oam_utils.c-> cspl: CSPL_StartModuleTimer
                            end
                            deactivate oam_utils.c
                        deactivate cfg_mom.c
                        end
                    deactivate cfg_mom.c
                deactivate cfg_mom.c
            end
            cfg_mom.c -> cfg_mom.c : publish_batch_result
                activate cfg_mom.c
                deactivate cfg_mom.c
        end
        deactivate cfg_mom.c
    end
end
deactivate oam_adp_layer.c
@enduml
