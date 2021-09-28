@startuml
|#pink|接收模块|
start
    fork

    :【接收】反向通道消息;
    :实体-adpLayerEntity-入口: oam_adp_layer_intf_process_msg;
    :oam_adp_frm_backward_msg\n构造list并入队,业务调用此接口向伪线程发送消息;

    note left
    1-将接收的CSPL插入到Node中
    2-将包插入到LIST-NODE中
    3-将LIST-NODE插入到Queue-Node中
    ====
    //LIST://
    typedef struct _List
    {
        NodeTag type;
        void *elem;
        struct _List *next;
    } <b>List;</b>
    //LIST-Node：//
    typedef struct {
        NodeTag type;
        Uint32 service;    /* 目标服务集 */
        Uint8 op;          /* 操作码 */
        Uint16 len;        /* 消息长度 */
        Uint8 *data;       /* 消息内容 */
    } <b>oam_adp_frm_backward_node;</b>
    //Queue-Node//
    typedef struct _QNode
    {
        YLNODE ynode;
        void *data;
    } <b>QNode;</b>
    end note
    :oam_adp_frm_backward_send_query_list\n进入发送队列;

    :enQueue\n发送通知;



    |#AntiqueWhite|转发模块|
    fork again

    :【发送】反向通道消息;
    :进程入口-初始化CSPL框架: oam_adp_frm_cspl_frame_init;
    :创建线程：oam_adp_frm_cspl_frame_thread\n初始化CSPL框架线程;
    :创建线程：oam_adp_frm_backward_thread\n创建反向通道消息发送线程;
    note left
    1-从Queue backward_channel_queue 解出 List *workQuery;
    workQuery->elem 携带的内容:
        typedef struct {
            NodeTag type;
            Uint32 service;    /* 目标服务集 */
            Uint8 op;          /* 操作码 */
            Uint16 len;        /* 消息长度 */
            Uint8 *data;       /* 消息内容 */
        } oam_adp_frm_backward_node;
    end note
    while(FRM_TRUE)
        :unQueue\n从队列中取出将要处理的QueryList, 可能阻塞;
        :foreach(queryNode, workQuery)\noam_adp_frm_send_backward_msg\n 遍历发送;
        note left
        从List *queryNode 解出 oam_adp_frm_backward_node
        end note
        :oam_adp_frm_clt_ucast_send\n异步通信客户端单播通信发送接口;
        note left
        msg: cspl-信息
        end note

        :oam_adp_frm_msg_sendup\n最终将消息发出;

        note left
        msg: g_frm_clt_client->send_buf
        msg: 组包数据：【帧头部】+【帧类型】+【帧信息】
        =====
        /* 帧头部 */
        #pragma pack (push)
        #pragma pack (1)
        typedef struct {
            Uint8 version : 4;       /* 版本号 */
            Uint8 dimention : 4;     /* 暂且保留 */
            Uint8 more_data : 1;     /* 暂且保留 */
            Uint8 reserved : 7;      /* 暂且保留 */
            Uint16 data_len;         /* 数据长度 */
        } oam_adp_frm_frame_head_t;
        #pragma pack (pop)

        /* 帧类型 */
        typedef struct {
            Uint8 reserved;         /* 暂且保留 */
            Uint8 service;          /* 服务集信息 */
            Uint8 op_code;          /* 操作码 */
        } oam_adp_frm_frame_type_t;

        /* 帧信息 */
        结构中的data, 即CSPL
        typedef struct {
            NodeTag type;
            Uint32 service;    /* 目标服务集 */
            Uint8 op;          /* 操作码 */
            Uint16 len;        /* 消息长度 */
            <b>Uint8 *data;</b>       /* 消息内容 */
        } oam_adp_frm_backward_node;

        end note
    endwhile
    fork again
    |#PowderBlue|发送模块|
    :CSPL_MAIN;
    :oam_adp_frm_cspl_frame_init\n初始化CSPL框架;
    :pthread_create(&ntid, NULL, oam_adp_frm_cspl_frame_thread, NULL)\n开启CSPL线程;
    :oam_adp_frm_cspl_frame_thread\n初始化CSPL框架线程;
    :oam_adp_frm_cspl_module_init\n初始化CSPL模块;
    :backward_channel_init\n初始化反向通道消息发送线程;
    :pthread_create(&serverId, NULL, oam_adp_frm_srv_init, NULL)\n异步通信服务器线程;
    :oam_adp_frm_srv_init\nadp框架异步服务器初始化
    :oam_adp_frm_srv_accept\n接收客户端连接;
    :oam_adp_frm_srv_recv\n接收消息;
    :<b>oam_adp_frm_srv_dispatch</b>\n接收完信息，进行消息调度处理;
    :oam_adp_frm_srv_ucast_recv\n处理每一个TLV消息;
    :(*g_frm_srv_ss[service].lte_oam_frm_ucast_recv)(client, p, frm_msg->msg_data, *frm_msg->msg_len)
    :<b>实际调用的是: ---------> adp_mt_ucast_rcv</b>;
    note left
    实际调用的是： adp_mt_ucast_rcv
    end note
    fork again
    |#DarkSeaGreen|注册模块(adp_mt_ucast_rcv)|
    :CSPL_MAIN;
    :oam_adp_frm_late_init;
    :adp_mt_late_init;
    :oam_adp_frm_srv_ucast_add_entry
    :g_frm_srv_ss[service].lte_oam_frm_ucast_recv = lte_oam_frm_ucast_recv;
    note left
        异步通信服务器接收入口添加接口
        * @service: 服务集
        * @adp_service_ucast_recv: 异步通信服务器接收入口函数
        * @client: 客户端信息
        * @op: 操作码
        * @msg: 消息缓存区指针
        * @len: 消息长度
    end note

    end fork
stop
@enduml
