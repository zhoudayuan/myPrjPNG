
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
    oam_crm_main.c -> oam_crm_ir_msg_procss.c  : oam_crm_mcb_crm_msg_process
    oam_crm_main.c --> oam_crm_ir_msg_procss.c : oam_crm_bpb_other_module_msg_process
end










box "CRM@OAM" #LightPink
participant  oam_crm.c
participant  oam_crm_cfg_smooth.c
end box

oam_crm.c -> oam_crm.c : oam_crm_process_bpb_msg //OAM-处理基带消息
ref over oam_crm.c
根据消息ID: OAM_CRM_RRU_SETUP_ABILITY_REQ
查表: g_bpbMsgToFuncTbl
调用: oam_crm_rru_setup_ability_noti_process
end ref
oam_crm.c -> oam_crm.c : oam_crm_rru_setup_ability_noti_process\n【处理基带板通知主控板RRU通道建立能力消息】
ref over oam_crm.c
参数: BBU上报CSPL参数
数据段:
RRU_CHANNEL_SETUP_ABILITY_STRU: RRU通道建立能力通知消息结构体
RRU_CHANNEL_SETUP_STRU: RRU通道建立请求结构定义
end ref

group  oam_crm_rru_setup_ability_noti_process\n处理基带板通知主控板RRU通道建立能力消息
    oam_crm.c -> oam_crm.c : oam_crm_get_rru_link_pos_by_cn\n根据链环编号获取链环全局索引号
    ref over oam_crm.c
        参数:CN: 链环编号:g_rruIdMapTbl[rruId - 1].rruCN
        返回值:RRU存储位置即链环全局索引
    end ref
    alt 如果组网方式为环型【RRU_LINK_RING_MODE】\n计算得出RRU_ID
        ref over oam_crm.c
            获取rruHPN【链环头光口号】
            获取方法: 根据BBU上报的RRU_ID, 查询映射关系表
            判断RRU为【主光口/备光口】接入
            判断方法: rruHPN 对比 g_rruCNTbl【RRU链环全局变量】
                          中的tPN/hPN【环头/尾光口号】
        end ref

        alt 主光口RRU接入
            oam_crm.c -> oam_crm.c : 计算主光口对应的rru id
        else 备光口RRU接入
            oam_crm.c -> oam_crm.c : 计算备光口对应的rru id
        end
    end
    oam_crm.c->oam_crm.c:光口状态fiberStatus设置为OFF\n映射关系表:g_rruIdMapTbl\nRRU单元全局变量:g_rruCfgTbl


    oam_crm.c->oam_crm.c: oam_crm_check_rru_is_exist_by_rruid\n判断RRU单元是否已经存在
    alt RRU单元存在
        ref over oam_crm.c
        赋值stRruVersionInf
        结构名称：【查询RRU版本校验结果响应结构体】
        结构类型： RRU_VER_CHECK_ACK_STR
        end
    else RRU单元不存在
    end


    oam_crm.c->OamSwmCtrledManager.c: oam_swm_rru_version_check\n调用软件管理模块函数获取RRU版本校验结果
    group oam_swm_rru_version_check\n校验RRU软件版本
        OamSwmCtrledManager.c->OamSwmCommon.c: oam_swm_get_runArea_swInfo
        OamSwmCtrledManager.c->OamSwmPkgUpdate.c: oam_swm_check_rru_verInfo
        OamSwmCtrledManager.c->oam_crm.c : SwmTrue
    end

    oam_crm.c -> oam_crm.c : 存储RRU通道建立能力\n该结构体类型:RRU_CHANNEL_SETUP_STRU \n该结构体变量: g_stRruChannelAbility[MAX_RRU_NUM]
    oam_crm.c -> oam_crm.c : oam_crm_active_mcb_crm_data_commit\nRRU通道建立数据更新，同步到备用主控板
    oam_crm.c -> oam_crm.c : oam_crm_send_bpb_crm_udp_msg
end
    oam_crm.c -[#Green]> oam_crm_main.c : OAM_CRM_RRU_SETUP_ABILITY_ACK_NACK

oam_crm_main.c -> oam_crm_ir_msg_procss.c : oam_crm_mcb_crm_msg_process
ref over oam_crm_main.c
    查询该表中的处理 g_msgToFuncTbl
    从主控板获取rru版本校验结果
    OAM_CRM_RRU_SETUP_ABILITY_ACK_NACK 对应
    oam_crm_rru_check_ack_msg_process
end ref



oam_crm_ir_msg_procss.c -> oam_crm_ir_msg_procss.c : oam_crm_rru_check_ack_msg_proces
oam_crm_ir_msg_procss.c ->  oam_crm_ir_msg_procss.c : oam_crm_ir_channel_create_conf_process\n组装通道接入配置相应消息并发送给RRU
oam_crm_ir_msg_procss.c ->  oam_crm_ir_msg_procss.c : oam_crm_send_msg_to_rru\n最终发送到RRU的接口
oam_crm_ir_msg_procss.c -[#Green]> RRU : CHANNEL_CREATE_CONFIGURE

@enduml
