@startuml
|#APPLICATION|1-timer--OAM_PM_SECOND_TIMEOUT_TYPE\nMR输出XML文件:mro,mre,mrs\n 入口: oam_mt_mr_report_meas_data|
start
    fork
        :PmEntity 定时器入口: \n oam_pm_intf_process_msg\n ;
        :noam_mt_mr_report_meas_data;
        note
            :mr(mro,mre,mrs)文件上报;
        endnote

        if (当前系统时间 >= 采样开始时间)  then (yes)
            :samp_start = 1 已经到达采样时间;
        endif

        if (上报时间是否到达) then (no)
            :退出(该时间递减);
        endif

        :上报时间到, 获取下次上报的秒数\noam_mt_get_next_pm_upd_sec;
        note
        参数-1: sampleBegin
        参数-2: sampleEnd
        参数-3: uploadPeriod
        参数-4: 输出:up_time_sec(下次上报的秒数)

        endnote

        partition 将测量数据输出到文件\noam_mt_mr_output_meas_file{
            :oam_mt_mr_output_meas_file\n将测量数据输出到文件;
            :获取enbid\n 填充 g_enb_info\n oam_mt_get_enb_info();
            note
            /* Static param struct of eNodeBConfiguration,基站配置 */
                typedef struct ST_S_EnodebCfg {
                    U32     enodebId; /* eNodeb标识 */
                    char    enodebName[20]; /* eNodeb名字 */
                } EnodebCfg_S_T;
            endnote

            :获取当前系统时间
                clock_gettime(CLOCK_REALTIME, &rt_time);
            :用于获取c_time时间的前一个采样时间，一般早于c_time\noam_mt_get_last_sample_time;
            split
                if (g_mr_task[task_id]->mro_type != 0) then(no)
                    :退出;
                endif


                partition mro文件输出\n入口:oam_mt_mr_output_mro_file {
                    :输出mro文件\noam_mt_mr_output_mro_file;


                    partition 把剩余的l3测量数据输出,节点删除的后面统一做\n入口:oam_mt_mr_proc_l3_smp_data {
                        :oam_mt_mr_proc_l3_smp_data\n一个采样周期结束后，将已经收到的L3采样数据进行处理;

                        if (在使能的前提下 task->mro_type 和 mrs_type 同时有效) then (no)
                        :退出;
                        endif

                        partition mro写入临时文件 {
                            :oam_mt_open_file_cont_write
                            打开文件
                            以继续写入方式打开文件，返回的指针偏移到文件尾;
                            note
                                入参:#define OAM_MT_TMP_FILE_NAME_STR    "%s/.%s_tmp.%u"
                            endnote
                        }
                        :将链表中数据(业务数据)写到临时文件
                        方法:遍历 task->mr_data.mro_l3_list
                        mro_l3_list是L3的数据
                        类型: mr_l3_data_node_t
                        list_for_each_entry_safe;
                        note
                            链表节点项(写入项):
                            typedef struct mr_l3_data_node_s {
                                    struct list_head list;
                                    struct timespec c_time;
                                    mr_o_e_report_info_t mr_l3_data;
                            } mr_l3_data_node_t;
                            typedef struct mr_o_e_report_info_s {
                                U8   mr_type;   /* mro:0,mre A1-A5:1-5*/
                                U32  recv_period_rpot_count;
                                mr_cell_info_t sc_cell_info;
                                U32  mme_s1_ue_id;
                                U32  mme_grp_id;
                                U32  mme_code;
                                U8   cell_id;
                                U8   nc_num;    /* ??? */
                                mr_cell_info_t nc_cell_info[OAM_MT_MR_NC_MAX];
                            } mr_o_e_report_info_t;//oam_mt_mr_l3_data_t;
                        endnote


                        if  (task->mro_type != 0) then (yes)
                            :oam_mt_ts_to_smp_prid_utc;
                            :oam_mt_create_eci;
                            :oam_mt_output_str_to_file;
                        endif
                        if  (task->mrs_type != 0) then (yes)

                            :oam_mt_mr_stat_rsrp
                            将g_mr_rsrp_stat写入到task;

                            :oam_mt_mr_stat_rsrq
                            将g_mr_rsrq_stat写入到task;
                        endif


                    }

                    partition "【MRO&MRE】"将临时输出的文件并入到正式文件中 {
                        :oam_mt_mr_write_final_file\n将临时输出的文件并入到正式文件中;
                        :oam_mt_open_new_file;
                        :oam_mt_open_file_rdonly;
                        :oam_mt_mr_output_meas_file_head;
                        :while(1) 读出和写入;
                        :oam_mt_output_str_to_file;
                        :oam_mt_mr_output_meas_file_footer;
                    }
                }
            split again
                partition "【mre】oam_mt_mr_output_mre_file" {
                    :把剩余的mre测量数据输出,节点删除的后面统一做
                    oam_mt_mr_proc_mre_smp_data(task_id);

                    :将零时输出的文件并入到正式文件中
                    oam_mt_mr_write_final_file(task_id, OAM_MT_TYPE_MRE, enb_id);
                }

            split again
                :oam_mt_mr_output_mrs_file;
            end split
        }



    fork again
    |#AliceBlue|2-timer:OAM_MT_MR_SAMP_TIMEOUT_TYPE\n一个采样周期到|

        :OAM_MT_MR_SAMP_TIMEOUT_TYPE 关联的动作;
        fork
            :执行;
            :oam_mt_mr_resp_smp_timer;
            partition 一个采样周期结束后，将已经收到的数据进行处理\n入口:oam_mt_mr_proc_smp_prd_data {
                :oam_mt_mr_proc_smp_prd_data;

                partition "获取 g_enb_info\n入口:oam_mt_get_enb_info" {
                    :oam_mt_get_enb_info();
                }


                partition "【L3-MR-处理(MRO/MRS)】\n一个采样周期结束后，将已经收到的L3采样数据进行处理\n写入到临时文件中 \n入口: oam_mt_mr_proc_l3_smp_data" {
                    :oam_mt_mr_proc_l3_smp_data(task_id)
                    遍历链表,取出task->mr_data.mro_l3_list作为SRC
                    类型：mr_l3_data_node_t;



                }


                partition  "【L2-MR-处理】\n一个采样周期结束后，将已经收到的L2采样数据进行处理" {
                    :oam_mt_mr_proc_l2_smp_data(task_id);
                    :memcpy(&l2_node, &g_l2_stat_data[i], sizeof(mr_l2_data_node_t));

                    :oam_mt_mr_stat_sinr(&l2_node, task)
                    更新task->mr_stat_data->sinr, 基于l2_node->mr_l2_data;

                    :oam_mt_mr_stat_rip(&l2_node, task);
                    :oam_mt_mr_stat_ripprb(&l2_node, task);
                    :oam_mt_mr_stat_phr(&l2_node, task);

                }

                partition "【MRE】一个采样周期结束后，将已经收到的mre数据进行处理\n入口:oam_mt_mr_proc_mre_smp_data" {
                    :oam_mt_mr_proc_mre_smp_data(task_id);
                }


                :oam_mt_mr_free_mre_list(task_id);
            }


        fork again
            :创建;
            note
                定时时间: 一个采样周期
                作用: 每周期处理上报的数据
            endnote


            split
                :proc 触发;
                :oam_mt_task_parse_mr_task;
                :启动定时器，每个采样周期结束后将数据写入临时文件等处理
                sample_ms = mr_task->task.SampleBeginTime * 1000 + mr_task->task.
                SamplePeriod;
                note
                    sample_ms 作为OAM_MT_MR_SAMP_TIMEOUT_TYPE 定时时间;
                endnote
            split again
                :timer触发;
                :oam_mt_mr_resp_smp_timer;
            endsplit
            partition "创建定时器: OAM_MT_MR_SAMP_TIMEOUT_TYPE" {
                :pm_start_unloop_timer;
                note
                    周期: 一个采样周期:sample_ms
                    返回值: mr_task->smp_timer
                endnote
                :pm_start_timer;
                :oam_start_new_timer;
            }
        end fork

    fork again
    |#AntiqueWhite|3-timer:OAM_MR_ACK_TIMER_TYPE|
        :OAM_MR_ACK_TIMER_TYPE;
        fork
            :执行;
            :对比 l3_cfg_status和l2_cfg_status;
            partition 查询当前状态信息 {
                :oam_mt_mr_measure_timeout_ack;
                note
                    :从接收到的消息中解出task_id;
                endnote
                while (遍历当前已有的小区)

                    partition L3判断 {
                        if (cfg_status[i] == MR_TASK_SENDING) (yes)
                            :cfg_status[i] == MR_TASK_ACK_TIMEOUT;
                            note
                                g_mr_task[task_id]->l3_cfg_status.cfg_status[i]
                            endnote
                        endif
                    }

                    partition L2判断 {
                    :l2_cfg_status 初始化地方-关联g_cell_info\noam_mt_task_send_mr_task_to_l2\拷贝函数oam_mt_get_cell_info\n;
                        if (l2_cfg_status.cfg_status[i] == MR_TASK_SENDING) (yes)
                            :cfg_status[i] = MR_TASK_ACK_TIMEOUT;
                            note
                                g_mr_task[task_id]->l3_cfg_status.cfg_status[i]
                            endnote
                        endif
                    }
                endwhile
            }
            partition oam_pm_mr_measure_timeout_ack {
                :oam_pm_mr_measure_timeout_ack;
                :OAM_SendMsgToNmsOrMML;
            }
        fork again
            :创建;
            split
                :oam_pm_mr_measure_mml_req;
            split again
                :oam_pm_mr_stop_timeout;
            split again
                :oam_mr_ni_resend_process;
            split again
                :oam_mt_task_send_mr_task_msg;
            end split
        end fork


    fork again
    |#Aquamarine|4-proc: MR_START_STOP_L3_ACK\n L3的返回结果|
        :MR_START_STOP_L3_ACK;
        note
            该消息的目的, 修改L3当前状态太信息
            状态枚举: oam_mt_mr_task_status_t
        endnote

        :oam_pm_intf_process_msg;
        :oam_mt_mr_process_l3_cfg_ack_msg;

        note
            接收L3的AKC消息
            rcv cfg ack from l3, cid=%d, ack=%d", cell_id, ack->result
            L3的消息格式
                typedef struct PmMrL3Ack
                {
                    U32 InstId;
                    U32 TaskId;
                    U8 CellId;
                    U8 result;
                    U8 src;
                    U8 reserve;
                } PmMrL3Ack_t;
        endnote

        :对比 cfg_status 是否为 MR_TASK_SENDING;
        note
            类型: oam_mt_mr_cfg_status_t
            变量: l3_cfg_status;
        endnote

        :根据L3返回的结果 (MR_L3_SUCCESS);
        :更新 g_mr_task[task_id]->l3_cfg_status.cfg_status[i];


    fork again
    |#Azure|5-proc: MR_START_STOP_L2_ACK\n L2的返回结果|
        :oam_mt_mr_process_l2_cfg_ack_msg;
        note
            typedef struct PmMrAck
            {
                U8 CellId;
                U8 result;
                U8 src;
                U8 reserve;
            } PmMrAck_t;
        endnote
        if (l2_cfg_status.cfg_status[i] == MR_TASK_SENDING) then (yes)
            :更新l2_cfg_status.cfg_status[i];
        endif



    fork again
    |#BUSINESS|6-proc: MR_L3_REPORT_DATA & MR_L2_REPORT_DATA\n功能:MR_L3_REPORT_DATA接收L2和L3上报的MR数据|
        :oam_mt_mr_intf_process_msg;
        split
        :oam_mt_mr_enqueue_l2_data\n解析CSPL存入 g_l2_stat_data;
        note
            SRC:
            CSPL的携带的数据: L2_Oam_Statistic_t
            typedef struct
            {
                //S16 nivalue[MAX_RB_NUMS];
                S16 nivalue[MAX_RB_NUMS] [MAX_SUBFRAME_NUM];  // MR.RIPPRB
                S16 wideband_nivalue[MAX_SUBFRAME_NUM];
                S8 sinrvalue[MAX_RB_NUMS];
                MacPhr_t phr[MAX_UE_NUM];//gid,416
                U16 phr_num;
                U16 cellID;//物理小区id
            }L2_Oam_Statistic_t;

            DST:
            本地数据: mr_l2_data_node_t
            typedef struct mr_l2_data_node_s {
                //struct list_head list;
                struct timespec c_time;
                Cell_S_T cell_info;
                L2_Oam_Statistic_t mr_l2_data;
            } mr_l2_data_node_t;
        endnote
        :将 g_cell_info 存入到 g_l2_stat_data;
        :copy(g_l2_stat_data[i].c_time, 系统时间)
        copy(g_l2_stat_data[i].cell_info, g_cell_info)
        copy(g_l2_stat_data[i].mr_l2_data, CSPL);


        split again
        :oam_mt_mr_enqueue_l3_data;
        endsplit

    fork again
    |#CadetBlue|7-proc:MT_TASK_MR_START\nMT_TASK_MR_STOP\nMT_TASK_MR_START\n网管使能|
        :oam_mt_task_parse_mr_task;
        :pm_stop_unloop_timer(mr_task->smp_timer);


        if (如果任务是enable，又接收到start任务) then (yes)
            :oam_mt_task_send_mr_task_msg(mr_task, task_type);
        endif




        :删除文件MRO,MRE\noam_mt_unlink_tmp_mr_file(new_task->InstId);

        partition 清空缓存中保存的数据 {
            :oam_mt_mr_free_mr_mear_data;
            :oam_mt_mr_free_l3_list;
            :oam_mt_mr_free_mre_list;
            :遍历清除oam_free(task->mr_stat_data[i]);
        }


        if (task_type == MT_TASK_MR_START) then (yes)
            :oam_mt_get_next_mr_upd_sec;
            :mr_task->smp_timer = pm_start_unloop_timer(OAM_MT_MR_SAMP_TIMEOUT_TYPE,
                sample_ms/* + OAM_MT_MR_SMP_DELAY_MS*/, (void *)smp_task_id_str, strlen(smp_task_id_str));

        elseif (task_type == MT_TASK_MR_STOP) then (yes)
            :mr_task->task.MrEnable
            pm_stop_unloop_timer(mr_task->smp_timer);
        else (no)
            :mr_task->task.MrEnable
            pm_stop_unloop_timer(mr_task->smp_timer);
        endif

        partition oam_mt_task_send_mr_task_msg {
            :oam_mt_task_send_mr_task_msg(mr_task, task_type);
            :oam_cfg_get_record_data_by_moid(mo_id, record_data);
            :oam_mt_get_cell_info
             获取 g_cell_info;


            partition "TO l2" {
                :oam_mt_task_send_mr_task_to_l2(task_type);
                note
                    上报数据: PmMrMeasure l2_meas;
                    类型
                        typedef struct {
                            U8  CellId;
                            U32 MrEnable;                   /* MR测量开关 */
                            U8  MrUrl[256];                 /* MR测量文件上传路径 */
                            U8  MrUsername[256];            /* 用户名 */
                            U8  MrPassword[256];            /* 密码 */
                            U32 MeasureType;                /* 测量类型 */
                            U8  OmaName[64];                /* OMC-R名称 */
                            U32 SamplePeriod;               /* 采样周期 */
                            U32 UploadPeriod;               /* 上报周期 */
                            U32 SampleBeginTime;            /* 采样开始时间 */
                            U32 SampleEndTime;              /* 采样结束时间 */
                            U8 PrbNum[256];                 /* 选择rb */
                            U8 SubFrameNum[32];             /* 选择子帧 */
                        }PmMrMeasure;
                endnote
                while (遍历g_cell_num)

                    partition "获取slot cell\入口:oam_mt_task_get_cell_index"{
                        :oam_mt_task_get_cell_index;
                        note
                            输入:cell
                            输出:slot proc
                            读redis
                            命令:"HMGET i.cell.%u slot proc", cell
                        endnote

                    }


                    :更新 l2_cfg_status.cell_id[i] = g_cell_info[i].cellId
                    更新 l2_cfg_status.cfg_status[i] = MR_TASK_SENDING;
                    :pm_send_L2_msg;
                    note
                    输入:
                    MR_START_STOP_L2_REQ,
                    &l2_meas
                    slot
                    proc
                    endnote
                end while
            }

            partition "TO l3" {
                :oam_mt_task_send_mr_task_to_l3(mr_task);
                note
                    send PmMrTaskMsg;
                endnote

                :oam_mt_task_get_cell_index;

                :CSPL_CONSTRUCT_HEADER(0,
                    OAM_MODULE_ID,
                    RRC_MODULE_Id,
                    MR_START_STOP_L3_REQ,
                    msg_len - CSPL_HEADER_SIZE,
                    msg)
                CSPL_SendMsg;

            }
        }
    fork again
    |#Bisque|7-配置:g_cell_info\nCell_S_T g_cell_info[3]|
        :g_cell_info;
        fork
            :获取;
            note
                /* Static param struct of Cell,小区 */
                typedef struct ST_S_Cell {
                    U32     cellId; /* 小区标识 */
                    char    sectorId[255]; /* 扇区索引 */
                    U32     phyCellId; /* 物理小区标识 */
                    U32     duplexingMode; /* 小区双工模式 */
                    U32     subFrameAssignment; /* 子帧配比 */
                    U32     specialSubframePatterns; /* 特殊子帧配比 */
                    U32     frameEdgeOffset; /* 空口帧边界调整量 */
                    U8      bandInd; /* 频带指示 */
                    U16     ulEarfcn; /* 上行频点 */
                    U16     dlEarfcn; /* 下行频点 */
                    U16     maxNumRRC; /* 最大可接入的连接态UE数 */
                    U32     bandwidth; /* 带宽 */
                    U32     antennaPortsCount; /* 天线端口数目 */
                    U32     antennaMode; /* 天线模式 */
                    U32     cellActiveState; /* 小区激活状态 */
                    char    plmnidx[255]; /* plmn索引 */
                    U8      taIdx; /* Ta索引 */
                    U32     timeAlignmentTimerCommon; /* 上行时间对齐定时器长度 */
                    U32     businessType; /* 商用场景 */
                    U32     maxUeNum; /* 用户规格 */
                    U32     rruMode; /* RRU模式 */
                    U32     dRXEnable; /* DRX使能开关 */
                } Cell_S_T;
            endnote
                :PROC;
                :oam_mt_task_intf_process_msg;
            split
                :PM;
                :oam_mt_task_parse_pm_task;
            split again
                :MR;
                :oam_mt_task_parse_mr_task;
                :oam_mt_task_send_mr_task_msg;
            split again
                :npt相关;
                :oam_mt_task_process_cell_add_msg;
            end split
            :oam_mt_get_cell_info;
            :oam_cfg_get_record_number_by_moid;
            note
                输入 DEF_MNO_CELL 207 /* 小区 */
            endnote
            :oam_cfg_get_record_data_by_moid(mo_id, record_data)\n查询mo记录的数据;

        fork again
            :使用;
        end fork
    fork again
        |#CornflowerBlue|8-获取:EnodebCfg_S_T g_enb_info\n基站配置|
        :EnodebCfg_S_T g_enb_info;

        note
            /* Static param struct of eNodeB Configuration,基站配置 */
            typedef struct ST_S_EnodebCfg {
                U32     enodebId; /* eNodeb标识 */
                char    enodebName[20]; /* eNodeb名字 */
            } EnodebCfg_S_T;
        endnote

        fork
            :创建;
            split
                :采样周期到，数据处理;
                :oam_mt_mr_proc_smp_prd_data;
            split again
                :输出XML;
                :oam_mt_mr_output_meas_file;
            end split

            partition  "获取 g_enb_info" {
                :oam_mt_get_enb_info;
                :oam_cfg_get_record_number_by_moid;
            }

        fork again
            :使用;
        end fork


    end fork
stop
@enduml
