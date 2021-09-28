@startuml
|#application|输出当前PM数据到XML中|
start
    fork
        :oam_mt_pm_output_stat_data;
        :oam_mt_open_new_file;
        partition  输出【文件头】 {
            :oam_mt_pm_output_meas_file_head;
            note right
                时间输出在此函数
            end note

            :oam_mt_ts_to_utc_tz;
            :oam_mt_output_str_to_file;
            note right
            写入的宏-OAM_MT_PM_MEAS_FILE_HEAD_STR
            end note
        }


    partition  输出【数据头】 {
        :oam_mt_pm_output_meas_data_head;
        :oam_mt_output_str_to_file;
        note right
        写入的宏-OAM_MT_PM_MEAS_DATA_HEAD_STR
        end note
    }


    partition  输出【测量信息】 {
        :oam_mt_pm_output_meas_info_head;
        :oam_mt_ts_to_utc_tz;
        :oam_mt_output_str_to_file;
        note right
        写入的宏-OAM_MT_PM_MEAS_INFO_HEAD_STR
        end note
    }


    partition  输出【测量类型】 {
        :oam_mt_pm_output_meas_type;
        :list_for_each_entry(meas_node, &g_meas_msg, list)\n{oam_mt_output_str_to_file};
        note right
        写入的宏-OAM_MT_PM_MEAS_TYPE_STR
        output_cnt
        meas_node->meas_name
        end note
    }



    partition  遍历当前任务表中的256个小区 {
    while (遍历表中256（OAM_MT_MAX_CELL_ID）个小区)
        if (当前小区测量数据是否已动态内存申请) then (yes))
            : 取出当前小区的测量数据;
        else (no)
            : 申请内存;
            while (遍历真实小区个数(g_cell_num))
                if (对比，获取当前小区个数，表中的该小区是否) then (yes)
                    :先申请表中的内存，再申请测量内存\n初始化;
                endif
            endwhile
        endif

        partition  输出【测量VAL头】 {
            : oam_mt_pm_output_meas_val_head;
            : oam_mt_create_eci;
            : oam_mt_output_str_to_file;
            note right
            写入宏OAM_MT_PM_MEAS_VALUE_HEAD_STR
            end note
        }

        while (遍历counter_id, 范围g_counter_total)
            if (根据counter_type的类型，选择操作数据库的方式) then (PmSet或PmAdd))
                :redis-hash-HGET\nKEY:counter_id和小区ID;
            else (PmSample或PmDelete)
                :redis-list-LRANGE\nkey:counter_id和小区ID 操作值：lrang_num;
            endif

            partition  将数据库的信息存储到任务任务结构中 {
                :oam_mt_pm_stat_meas_data\n将redis中的数据写入到任务表中;
                note right
                    参数-1：reply,  存储的源
                    参数-2：&p[j]： 存储的目的
                    参数-3： &g_countid_arr[j]：作为筛选的条件
                    任务表: p = *(g_pm_task[task_id]->meas_data[i])
                    内存单元：p = (oam_mt_pm_data_t *)oam_malloc(sizeof(oam_mt_pm_data_t) * g_counter_total)
                    内存单元-info:
                            typedef struct oam_mt_pm_per_data_s {
                                UL64 meas_data; //<----写入的位置
                                U16  stat_times;
                                U16  lrang_num;
                            } oam_mt_pm_data_t;
                end note

                if (根据counter_type的类型，选择存储方式) then (PmSet或PmAdd))
                    if (val < data->meas_data) then (yes)
                        :data->meas_data = 0;
                    else (no)
                        :data->meas_data = val - data->meas_data;
                    endif
                    :data->stat_times = 0;
                else (PmSample或PmDelete)
                    :一次性从数据库中读出多个值
                    即处理redis列表操作;
                    if (stat_type) then (最大值)
                        :如果data->meas_data < val 则data->meas_data = val
                        data->stat_times = 1;
                        note right
                            STAT_TYPE_DER_MAX-以DER方式进行统计，获取该时间段内的最大值
                            STAT_TYPE_SI_MAX-以si方式进行统计，获取该时间段内的最大值
                        end note

                    else (平均值)
                        :累加
                        data->meas_data += val
                        data->stat_times++;
                        note right
                            STAT_TYPE_DER_AVG = 0,  /* 以DER方式进行统计，获取该时间段内的平均值 */
                            STAT_TYPE_DER_CNT,      /* 以DER方式进行统计，获取该时间段内的合计值 */
                            STAT_TYPE_SI_AVG,       /* 以si方式进行统计，获取该时间段内的平均值 */
                            STAT_TYPE_CC,
                            STAT_TYPE_GAUGE
                        end note
                    endif
                endif
            }
        endwhile

        while (遍历测量项， 从链表 g_meas_msg 取出)
            : 当前测量项，重点关注该测量项有多少个counter_id;
                note right
                    typedef struct oam_mt_pm_meas_info_s {
                            struct list_head list;
                            char meas_name[OAM_MT_PM_MEAS_NAME_STR_LEN];      /* 测量项目名称 */
                            char meas_obj[OAM_MT_PM_MEAS_OBJ_NAME_STR_LEN];   /* 测量对象名称 */
                            U32  counter_num;
                            U32  out_type;
                            U32  counter_id[OAM_MT_PM_MAX_COUNTID_NUM];     /* 测量项对应的id */
                            U32  counter_type[OAM_MT_PM_MAX_COUNTID_NUM];   /* 测量项对应的id */
                        oam_mt_pm_stat_type_t stat_type;                    /* 统计的方式 */
                    } oam_mt_pm_meas_info_t;`
                endnote

                partition  该测量项所有counter_id对应的值存入到out_val {
                    while (遍历当前测量项的counter_id， counter_num)
                        if (判断当前counter_id的stat_type) then (平均值)
                            if (当前counter_id的 stat_times == 0)  then (yes)
                                :out_val += (F64)p[stat_cnt].meas_data;
                            else (no-计算平均值)
                                :out_val += ((F64)p[stat_cnt].meas_data / p[stat_cnt].stat_times);
                                note right
                                    此时stat_times 对应 reply->elements
                                endnote
                            endif
                        else (最大值 累加值)
                            :out_val += p[stat_cnt].meas_data;

                            note right
                                p: 从redis中取出的数据
                                stat_cnt: 是当前测量项的第几个counter_id
                            endnote
                        endif
                    endwhile
                }

                :根据out_type的类型设置打印格式;
                note right
                    输出数据类型（outType）：
                    0=实数
                    1=整数
                    2=如果无此值则输出‘-’，否则实数
                    3=如果无此值则输出‘-’，否则整数
                    endnote
                :oam_mt_output_str_to_file\n写入文件;
            endwhile
        endwhile
        :输出结尾字符串: oam_mt_output_str_to_file(fd, OAM_MT_PM_MEAS_VALUE_END_STR);
        :输出-INFO_FOOTER: oam_mt_output_str_to_file(fd, OAM_MT_PM_MEAS_INFO_FOOTER_STR);
        :输出-DATA_FOOTER: oam_mt_output_str_to_file(fd, OAM_MT_PM_MEAS_DATA_FOOTER_STR);
        :输出页脚-footer-oam_mt_pm_output_meas_file_footer;
        :释放内存-oam_mt_pm_free_meas_data;
        :初始化测量数据-oam_mt_pm_init_meas_data;
        :上传通知：oam_mt_task_notify_upload_file;
    }







    fork again
    |#DARKKHAKI|接口:批量加:oam_pmm_add_batch_counters|
        :uuuuuuuuuuuuuuuuuuuuuuuuuuu;
        :uuuuuuuuuuuuuuuuuuuuuuuuuuu1;
        :uuuuuuuuuuuuuuuuuuuuuuuuuuu3;
    endfork
stop
@enduml
