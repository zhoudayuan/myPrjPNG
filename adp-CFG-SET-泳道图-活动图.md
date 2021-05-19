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
participant  mapping.c
participant  oam_crm_main.c
participant  oam_utils.c
collections  timer@adp
end box
participant  cspl


timer@adp->timer@adp: msg_timer_out_proc

timer@adp->cfg_mom.c : process_some_request
activate cfg_mom.c
    cfg_mom.c->cfg_mom.c : process_request
    activate cfg_mom.c
        cfg_mom.c->mapping.c: oam_transfer_value
        activate mapping.c
        deactivate mapping.c
    deactivate cfg_mom.c
deactivate cfg_mom.c

tr069_agent -[#Green]> oam_adp_tr069.c : TR069_SET_PARAM(api_id<0x1535>)
note left
CSPL-携带消息:
typedef struct{
	U32 data_path_num;
	set_data_path_t data_path[DATA_PATH_MAX];
}tr069_set_t;
typedef struct{
	char path[256];
	U32  value_len;
	char value[256];
}set_data_path_t;
eg:Device.LteConfig.Private.LTE.S1C.0.LocIpAddrList
end note

oam_adp_tr069.c -> oam_adp_tr069.c: \n解析RJ/H3的消息,转换成adp认识的消息\ntr069_process_set_msg

activate oam_adp_tr069.c
group 函数实现:tr069_process_set_msg\n参数-1:msg-去掉cspl头的消息,RJ/H3的消息, \n参数-2:trans_id-消息ID, \n参数-3:len-消息长度(CSPL头中的长度，即携带消息的长度)
    oam_adp_tr069.c -> oam_adp_tr069.c
    activate oam_adp_tr069.c
    deactivate oam_adp_tr069.c
end
deactivate oam_adp_tr069.c

oam_adp_tr069.c -> oam_adp_layer.c: 0x1fff: V_LTEOAM__OPERATE_TYPE_E__SET_VALUE

note left
src:TR069_TASK_MODULE_ID
dst:OAM_ADP_MODULE_ID
CSPL-携带消息:
    typedef struct _adp_cfg_convert_t {
        U32  oper_type;
        U32  oper_id;
        U32  param_name_len;
        char param_name[256];
        U32  n_id_array;
        U32  id_array[32];
        int  value_len;
        char value[256];
    } adp_cfg_convert_t;
end note

oam_adp_layer.c -> oam_adp_layer.c: oam_adp_layer_intf_process_msg
activate  oam_adp_layer.c
    oam_adp_layer.c-> cfg_mom.c : oam_adp_tr069_req_process
    activate   cfg_mom.c
    alt  V_LTEOAM__OPERATE_TYPE_E__SET_VALUE
        cfg_mom.c->mapping.c : SET操作需要等待END消息后再处理\noam_get_seq_id
        cfg_mom.c->mapping.c : oam_merge_mo_oper
        activate mapping.c
        deactivate mapping.c

    end
    deactivate cfg_mom.c
deactivate  oam_adp_layer.c





@enduml
