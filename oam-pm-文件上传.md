@startuml
|#pink|接收模块|
start
:PmEntity:oam_pm_intoam_pm_intf_process_msg\nPM实体定时器入口;
:case: OAM_PM_SECOND_TIMEOUT_TYPE \n通用1s定时器;
:oam_mt_pm_report_meas_data\n进入到PM上报处理;
:找到当前使能任务;
:oam_mt_get_next_pm_upd_sec\n获取下次上报的秒数;
stop
@enduml
