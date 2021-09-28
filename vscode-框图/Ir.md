
@startuml


RRU -[#Green]> oam_crm_main.c      : CHANNEL_CREATE_REQUEST
note left
消息头:
typedef struct IR_IE_HEADER
{
    U32 ulMsgId;  //消息编号
    U32 ulMsgLength;//消息长度
    U8 ucRruId;//逻辑编号
    U8 ucBbuId;//逻辑编号
    U8 ucFiberId;//光纤端口号
    U32 ulTransId;//流水号
}IR_IE_HEADER_STRU;
消息体:
RRU_PRODUCT_IE_STRU //解包RRU产品IE
CHANNEL_CREATE_REASON_IE_STRU //解析通道建立原因IE
RRU_CAPACITY_IE_STRU // 解析rru能力IE
RRU_CASCADE_IE_STRU // 解析RRU级数IE
RRU_HW_VERSION_INFO_IE_STRU // 解析RRU硬件类型及版本信息IE
RRU_SW_VERSION_INFO_IE_STRU // 解析RRU软件版本信息IE
RRU_BAND_CAPACITY_IE_STRU // RRU频段能力IE
RRU_RADIO_CHANNEL_IE_STRU // RRU射频通道能力IE
CARRIER_ABILITY_IE_STRU // 载波能力组合IE
end note


box "CRM@AGENT" #LightBlue
participant  oam_crm_main.c
participant  oam_crm_ir_msg_procss.c
participant  oam_crm_unpack_ie_message.c

end box


alt 无CSPL头
    oam_crm_main.c -> oam_crm_ir_msg_procss.c : oam_crm_ir_message_process
    oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c: ulRet = oam_crm_channel_create_process
    activate oam_crm_ir_msg_procss.c
    loop do-while
        alt RRU_PRODUCT_IE_ID: //RRU产品IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_product_ie
        else CHANNEL_CREATE_REASON_IE_ID: //通道建立原因IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_channel_create_reason_ie
        else RRU_CAPACITY_IE_ID // rru能力IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_capacity_ie
        else RRU_CASCADE_IE_ID //RRU级数IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_cascade_ie
        else RRU_HW_VERSION_INFO_IE_ID //RRU硬件类型及版本信息IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_hw_version_info_ie
        else RRU_SW_VERSION_INFO_IE_ID //RRU软件版本信息IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_sw_version_info_ie
        else RRU_BAND_CAPACITY_IE_ID //RRU频段能力IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_band_capacity_ie
        else RRU_RADIO_CHANNEL_IE_ID //RRU射频通道能力IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_rru_radio_channel_ie
        else CARRIER_ABILITY_IE_ID //载波能力组合IE
            oam_crm_ir_msg_procss.c -> oam_crm_unpack_ie_message.c:oam_crm_unpack_carrier_ability_ie
        end
    end
    note left
        将RRU上报的消息写入到 g_stRruChannelAbility
        /*RRU通道建立能力结构体*/
        RRU_CHANNEL_SETUP_STRU g_stRruChannelAbility[MAX_RRU_NUM];
    end note
    deactivate oam_crm_ir_msg_procss.c
    alt ulRet=成功
        oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c :oam_crm_send_rru_setup_ability_req_msg
        oam_crm_ir_msg_procss.c -[#Green]> oam_crm.c  : OAM_CRM_RRU_SETUP_ABILITY_REQ\nRRU通道建立能力通知消息
        note right
            g_stRruChannelAbility 写入到 g_stRruChannelAbility 上报 OAM
            /*RRU通道建立能力结构体*/
            RRU_CHANNEL_SETUP_STRU g_stRruChannelAbility[MAX_RRU_NUM];
        end note
    else ulRet=失败
        oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c : 返回失败数值
    end
else 有CSPL头
    oam_crm_main.c -> oam_crm_ir_msg_procss.c : oam_crm_mcb_crm_msg_process
    oam_crm_main.c -> oam_crm_ir_msg_procss.c : oam_crm_bpb_other_module_msg_process
end



box "CRM@OAM" #LightPink
participant  oam_crm.c
participant  oam_crm_cfg_smooth.c
participant  OamSwmCtrledManager.c
participant  OamSwmCommon.c
participant  OamSwmDbs.c
participant  OamSwmPkgUpdate.c
participant  oam_cfg.c
participant  oam_dev_state_control.c

end box

oam_crm.c -> oam_crm.c : oam_crm_process_bpb_msg //OAM-处理基带消息
ref over oam_crm.c
根据消息ID: OAM_CRM_RRU_SETUP_ABILITY_REQ
查表: g_bpbMsgToFuncTbl
调用: oam_crm_rru_setup_ability_noti_process
end ref


    group  函数: oam_crm_rru_setup_ability_noti_process\n功能:处理基带板通知主控板RRU通道建立能力消息\n入参:BBU上报CSPL消息\n\tCSPL携带的信息:RRU_CHANNEL_SETUP_ABILITY_STRU【RRU通道建立请求】

        oam_crm.c -> oam_crm.c : 取出BBU上报消息的【rruID】\n作为索引,以该索引通过【g_rruIdMapTbl】找到【链环编号】\n根据链环编号获取【链环全局索引号】\n将g_rruIdMapTbl、g_rruCfgTbl全局中的对应RRU的光口状态置为ON\n调用函数\noam_crm_get_rru_link_pos_by_cn\n获取【链环全局索引号】即【RRU存储位置即链环全局索引】

        alt 如果组网方式为环型【RRU_LINK_RING_MODE】\n计算备光口对应的rru id
            ref over oam_crm.c
                获取rruHPN【链环头光口号】
                获取方法: 根据BBU上报的RRU_ID, 查询映射关系表【g_rruCNTbl】
                判断RRU为【主光口/备光口】接入
                判断方法: rruHPN 对比 g_rruCNTbl【RRU链环全局变量】中的tPN/hPN【环头/尾光口号】
            end ref
            oam_crm.c -> oam_crm.c : rruHPN【链环头光口号】和tPN【环尾光口号】hPN【环头光口号】对比
            alt rruHPN==tPN, 主光口RRU接入
                oam_crm.c -> oam_crm.c : 计算主光口对应的rru id
            else rruHPN==hPN, 备光口RRU接入
                oam_crm.c -> oam_crm.c : 计算备光口对应的rru id
            end
            oam_crm.c->oam_crm.c:光口状态: \n在g_rruIdMapTbl和g_rruCfgTbl中fiberStatus设置为OFF
        end

        alt oam_crm_check_rru_is_exist_by_rruid\n判断RRU单元是否已经存在
            oam_crm.c -> oam_crm.c : 从【g_rruCfgTbl】中获取版本信息保存在【stRruVersionInf】
        else RRU单元不存在,返回
        end

        oam_crm.c->OamSwmCtrledManager.c: oam_swm_rru_version_check\n调用软件管理模块函数获取RRU版本校验结果

        group 函数: oam_swm_rru_version_check\n作用:校验RRU软件版本\n输入参数:stRruVersionInf-的柜框槽号及对应的硬件版本号、软件版本号\n输出参数:pVerifyResult-版本校验结果
            OamSwmCtrledManager.c->OamSwmCommon.c : oam_swm_get_runArea_swInfo
            group 函数:oam_swm_get_runArea_swInfo\n1.根据老的hardId映射为新的hardId\n2.根据新的hardId获取运行区的版本信息
                OamSwmCommon.c->OamSwmDbs.c:oam_swm_db_get_brd_ver_info\n使用新方式获取该基带板的ftp信息
                group 函数: oam_swm_db_get_brd_ver_info \n内部实现: 读取redis中的SW_INFO\n功能:获取主区单板版本信息(需要外部释放内存)\n\t1.根据包类型、模式、状态获取包索引\n\t2.根据硬件id获取单板版本信息
                    OamSwmDbs.c->OamSwmDbs.c :oam_swm_connect_redis_server
                end
            end
            OamSwmCtrledManager.c->OamSwmPkgUpdate.c : oam_swm_check_rru_verInfo
            group 函数:oam_swm_check_rru_verInfo 内部实现\n功能:通知虹信下载且升级\n参数-1:<输入>*rruVerInfo，BBU上报数据的一份拷贝\n参数-2:<输入>swNum, 从redis中返回的数据\n参数-3:<输入>*tmpSwInfo,redis中读取的数据\n参数-4:<输出> *rruUpdateInfo\n
                loop swNum次(由参数传入)
                    alt SwType == SWM_VERSION_TYPE_APP
                    OamSwmPkgUpdate.c->OamSwmPkgUpdate.c: A
                    else SwType == SWM_VERSION_TYPE_SYS
                    OamSwmPkgUpdate.c->OamSwmPkgUpdate.c: A
                    end
                end

                alt updateFlag==SWM_UPDATING
                    OamSwmPkgUpdate.c->OamSwmPkgUpdate.c: 只要有一种软件类型版本不一致,则要置为updateing

                else updateFlag==SWM_NOT_UPDATING
                end

            end


            OamSwmCtrledManager.c->oam_crm.c : SwmTrue
        end
        note right
            主要判断函数
        end note

        oam_crm.c -> oam_crm.c : 获取ftp地址
        ref over oam_crm.c
            MCB_OAM_IP  "10.11.1.130"
            该地址保存在【rruVerCheckAck】中返回给BBU
        end ref

        oam_crm.c -> oam_crm.c : 存储RRU通道建立能力
        ref over oam_crm.c
            将BBU上报的数据保存在【g_stRruChannelAbility】中
        end ref


        group 主用主控板同步crm全局变量信息到备用主控板
            oam_crm.c -> oam_crm.c : oam_crm_active_mcb_crm_data_commit
                group oam_crm_active_mcb_crm_data_commit\n提交备板
                oam_crm.c->oam_cfg.c :oam_cfg_get_ha_cfg_sync_status(返回gCfgSyncDataFlag)
                alt gCfgSyncDataFlag==SYNC_OK
                    ref over oam_crm.c
                        将【cspl_ha_record_t haData】
                        提交一条记录到备板，作主的时候使用
                    end ref
                end
            end
        end
        oam_crm.c -> oam_crm.c:oam_crm_disp_rru_channel_ability\n显示RRU信道能力信息
            ref over oam_crm.c
                g_stRruChannelAbility保存RRU本次上报的信息
                以BBU上报的RRU_ID作为数组g_stRruChannelAbility的索引值
                用作oam_crm_disp_rru_channel_ability的参数
                打印输出
            end ref

        group 向基带板发送RRU版本校验结
            oam_crm.c -> oam_crm.c : oam_crm_send_bpb_crm_udp_msg
            group 发送UDP消息响应到BPB CRM
                oam_crm.c -> oam_crm.c : oam_crm_judge_bpb_available\n判断基带板是否可用
                group 从状控获取可用的基带板的个数及模块号
                    oam_crm.c -> oam_dev_state_control.c : get_bpb_oam_agt_ipc
                end
            end
        end

    end
    oam_crm.c -[#Green]> oam_crm_main.c : OAM_CRM_RRU_SETUP_ABILITY_ACK_NACK

oam_crm_main.c -> oam_crm_ir_msg_procss.c : oam_crm_mcb_crm_msg_process
ref over oam_crm_main.c
    查询该表中的处理 g_msgToFuncTbl
    从主控板获取rru版本校验结果
    OAM_CRM_RRU_SETUP_ABILITY_ACK_NACK 对应
    oam_crm_rru_check_ack_msg_process
end ref

oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c : oam_crm_rru_check_ack_msg_process
oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c : oam_crm_ir_channel_create_conf_process\n组装通道接入配置相应消息并发送给RRU
oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c : oam_crm_send_msg_to_rru\n最终发送到RRU的接口
oam_crm_ir_msg_procss.c -[#Green]> RRU : CHANNEL_CREATE_CONFIGURE

@enduml
