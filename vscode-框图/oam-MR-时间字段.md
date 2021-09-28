@startuml

|#SandyBrown|PM结构:oam_mt_pm_task_t\记录还有多少秒上报\nnup_time_sec\n关键函数:oam_mt_get_next_pm_upd_sec|
start
    fork
        :更新-up_time_sec;
        split
        :proc\n新任务解析更新\noam_mt_task_parse_pm_task;
        split again
        :timer\n周期更新 up_time_sec递减\noam_mt_pm_report_meas_data;
        endsplit
        :oam_mt_get_next_pm_upd_sec\n获取下次上报的秒数;

    fork again
    |#PaleVioletRed|PM结构:oam_mt_pm_task_t\n一个上报周期内的首次采样时间\nfirst_intl_smp_ts|
        split
            :更新-first_intl_smp_ts;
            split
                :异步\n新任务来临时\noam_mt_task_parse_pm_task;
                :更新（初始化）为当前系统时间\npm_task->first_intl_smp_ts.tv_sec = ts_r_cur.tv_sec\n同时也更新结束时间;
            split again
                :同步:输出XML之后\noam_mt_pm_report_meas_data;
                :一是因为oam_mt_get_next_upd_sec的时候做了+1处理
                g_pm_task[i]->first_intl_smp_ts.tv_sec = rt_time.tv_sec - 1;
            endsplit
        split again

        :作用;
        :oam_mt_pm_report_meas_data;
        :oam_mt_pm_output_stat_data;
        :输出时间在当前文件中-beginTime\noam_mt_pm_output_meas_file_head;
        endsplit


    fork again
    |#Yellow|PM结构:last_intl_smp_ts\n一个上报周期内的末次采样时间\n字段:last_intl_smp_ts|
    split
        :作用;
        :oam_mt_pm_report_meas_data;
        :输出当前数据\noam_mt_pm_output_stat_data;
        split
            :oam_mt_pm_output_meas_file_footer;
        split again
            :oam_mt_pm_output_meas_info_head;
        endsplit
    split again
        :更新;
        split
            :timer;
            :oam_mt_pm_report_meas_data;
            :oam_mt_pm_output_stat_data;
        split again
            :proc;
            :oam_mt_task_parse_pm_task;
            :pm_task->last_intl_smp_ts.tv_sec =\n ts_r_cur.tv_sec + pm_task->up_time_sec - 1;
        endsplit
    endsplit


    fork again
    |#Aquamarine|MR结构:oam_mt_mr_task_t\n记录还有多少秒上报\n字段:up_time_sec|
    :更新;
    split
        :timer;
        :oam_mt_mr_report_meas_data;

    split again
        :proc;
        :oam_mt_task_parse_mr_task;
    endsplit

    partition "获取下次上报剩余的秒数\n入口: oam_mt_get_next_mr_upd_sec" {
        :oam_mt_get_next_mr_upd_sec;
        note
            开始: SampleBeginUtcTime
            结束: SampleEndUtcTime
            周期: UploadPeriod
            计算出【记录还有多少秒上报】: up_time_sec
        endnote
        :计算【上报时间】与【当前时间的距离】\ntm_interval = tm_begin - tm_local;
        if (tm_interval>0) then (yes)
            :upload_time = tm_begin + upload_period;
        else (no)
            :num = -(-(tm_interval) % upload_period);
        endif
    }


    fork again
    |#LightSteelBlue|oam_mt_get_next_mr_upd_sec|

        :11111111111111111111;
            :oam_mt_report_meas_data\n 到了上报周期时，需要输出mro、mre、mrs文件;
            note
                【up_time_sec】周期递减时间到上报;
            endnote


    fork again
        |#PowderBlue|【SampleBeginUtcTime】\n1-网管下发参数\n2-决定什么时间开始采样数据|
        :oam_mt_mr_report_meas_data\n;
        :oam_mt_cmp_tm(cur_tm, &(g_mr_task[i]->task.SampleBeginUtcTime)) \n当前时间做差比较关联--->【samp_start】;

    fork again
        |#pink|【samp_start】与业务相关|
        :samp_start与SampleBeginTime 关联， 当SampleBeginTime为0时候，要立即使能;
    fork again

        |#AntiqueWhite|【SampleBeginTime】采样开始时间：\n|
        :oam_mt_task_parse_mr_task\n【SampleBeginTime 初始值】 与 up_time_sec 和 UploadPeriod 关联;
    fork again
        |#PowderBlue|【SamplePeriod】|

    end fork
stop
@enduml

