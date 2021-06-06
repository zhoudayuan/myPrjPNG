@startuml
|#pink|接口：oam_pmm_init \nagent获取pmm共享内存首地址: gKpiHead|
start

    fork
        :bpbPmmEntity---实体;
        :pm_bpb_init;
        :pm_attach_kpi_shm;

        partition 获取共享内存首地址 {
            :pm_ose_attach_shm;
            note left
                gKpiHead = (PmBpbShmHead *)pm_ose_attach_shm("PMM_SHM_BASE", &size)
                gKpiHead: 共享内存首地址
                gKpiHead 的类型
                        typedef struct PmBpbShmHead {
                            U32 totalSize;
                            U32 freeOffset;
                        } PmBpbShmHead;
            endnote
            :oss_get_shared_block;
            note left
                * oss_get_shared_block - allocate block shared memory from soc_shared area.
                * @name: the name of specified block shared memory.
                * @type: the type of specified block shared memory.
                *		CPU_CPU indicates memory could be shared between cpu cores.
                *		CPU_DSP indicates memory be shared between cpu and dsp only.
                *		CPU_DSP as default value.
                * @addr: output parameter, the logical address of block shared memory allocated.
                * @size: output parameter, the size of block shared memory allocated.
                * return value - error code from ose_memory_mgr
            endnote
        }

    fork again
        |#LightGray|接口:批量加:oam_pmm_add_batch_counters|
        :oam_pmm_add_batch_counters;
        note
            L2-传入的参数的结构:
            //批量操作计数器需保证有相同Level
            typedef struct PmmCounterBatchHandler {
                U32 beginCounterId;
                U32 counterCnt;
                PmmCounterLevel counterLevel;
                PmmCounterKey key;
            } PmmCounterBatchHandler;
        endnote


        partition  PmAdd操作 {
            :pm_batch_cmds;

            :PmCounterCmd *cmd = (PmCounterCmd *)(gKpiEnt + gKpiHead->freeOffset);
            note
                gKpiEnt 是去头之后的地址
                gKpiEnt = (U8 *)(gKpiHead + 1);
                L2写入共享内存中的数据：
                typedef struct PmCounterCmd{
                    PmCounterCmdType cmd;
                    U8 key[PM_KEY_LEN];
                    UL64 counterId;
                    UL64 cnt;
                    UL64 reserve;
                } PmCounterCmd;
            endnote

            :根据入参的 counterCnt 往共享内存中写数据
            counterCnt来自L2的参数  PmmCounterBatchHandler;
        }


    fork again
        |#MediumAquaMarine|接口:单独加:oam_pmm_add_counter|
        :oam_pmm_add_counter;
        :pm_cmd:参数:PmAdd;
        :pm_batch_cmds;


    fork again
        |#Sienna|接口:设置:oam_pmm_set_batch_counters|
        :pm_batch_cmds:参数:PmSet;

    fork again
        |#DodgerBlue|接口:设置: oam_pmm_set_counter|
        :pm_cmd:参数:PmSet;
        note
            PmmCounterHandler 转 PmmCounterBatchHandler
            【SRC】:
            typedef struct PmmCounterHandler
            {
                U32 counterId;
                PmmCounterLevel counterLevel;
                PmmCounterKey key;
            } PmmCounterHandler;

            【DST】:
            //批量操作计数器需保证有相同Level
            typedef struct PmmCounterBatchHandler {
                U32 beginCounterId;
                U32 counterCnt;
                PmmCounterLevel counterLevel;
                PmmCounterKey key;
            } PmmCounterBatchHandler;
        endnote

        :pm_batch_cmds;
        note
            【DST】:
            //批量操作计数器需保证有相同Level
            typedef struct PmmCounterBatchHandler {
                U32 beginCounterId;
                U32 counterCnt;
                PmmCounterLevel counterLevel;
                PmmCounterKey key;
            } PmmCounterBatchHandler;

            【SHM】
            typedef struct PmCounterCmd{
                PmCounterCmdType cmd;
                U8 key[PM_KEY_LEN];
                UL64 counterId;
                UL64 cnt;
                UL64 reserve;
            } PmCounterCmd;
        endnote


    fork again
        |#AntiqueWhite|2-agent收到L2读共享内存消息通知\n 然后转发给OAM\n接收消息：PM_SHM_PREPARE_DATA_ACK, \n转发消息：PM_AGENT_SEND_COUNTER |
        :deal_shm_prepare_data_ack;
        :pm_bpb_transmit_cmds;
        :pm_bpb_send_msg_to_mcb;

        note left
            agent的功能：
                【转发消息】：接收到PM_SHM_PREPARE_DATA_ACK后
                组PM_AGENT_SEND_COUNTER包通知 OAM
                【消息的内容】：共享内存数据，PM信息，由L2写入
        endnote

    fork again
        |#LightGreen|OAM接收PM_AGENT_SEND_COUNTER，\nOAM将消息写入数据库|

        :oam_pm_intf_process_msg:
            case: PM_AGENT_SEND_COUNTER;
        :pm_counter_from_L2;
        :pm_init_handlerne\n获取数据库句柄，通过Enodeb去读取数据库， Enodeb=1;


        partition 解析CSPL消息，写入数据库 {
            :pm_counter_multi_cmd;
            note left
                1-遍历CSPL消息
                2-自接续颠倒
                3-写入数据库
            endnote

            :ntoh_PmCounterCmd 字节序颠倒;
            :append_cmd;
        }
    endfork





stop

@enduml
