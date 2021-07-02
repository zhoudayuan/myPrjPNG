@startjson
{
    "message_module":[
        {"messageID":5152,"module":0},
        {"messageID":5153,"module":0},
        {"messageID":5154,"module":0},
        {"messageID":5155,"module":0},
        {"messageID":5156,"module":1},
        {"messageID":5157,"module":0},
        {"messageID":5158,"module":0},
        {"messageID":5159,"module":0},
        {"messageID":5160,"module":0},
        {"messageID":5161,"module":0},
        {"messageID":5162,"module":0},
        {"messageID":5163,"module":0},
        {"messageID":5164,"module":0},
        {"messageID":5165,"module":0},
        {"messageID":5166,"module":0},
        {"messageID":5167,"module":0},
        {"messageID":5168,"module":0},
        {"messageID":5173,"module":0}
        ],
    "messageIdModuleNum":18,

    "complex_cmd":[
        {"moid":207,"opCode":0},
        {"moid":207,"opCode":1}
        ],
    "complexCmdNum":2,

    "addList":[
        {"mo_name":"MceCfg","mo_id":700,"moCat":27},
        {"mo_name":"M3Interface","mo_id":701,"moCat":27},
        {"mo_name":"CellCfg","mo_id":702,"moCat":27},
        {"mo_name":"CsaCfg","mo_id":704,"moCat":27},
        {"mo_name":"MaCfg","mo_id":703,"moCat":27},
        {"mo_name":"MCEDBInfo","mo_id":799,"moCat":27}
        ],
    "addListNum":6,

    "mapTbl_name":[
        {"tableId":700,"tableName":"MO_MCECFG","paramNum":6,"allParamNum":6,"belongCmd":"MOD,QRY"},
        {"tableId":701,"tableName":"MO_M3INTERFACE","paramNum":2,"allParamNum":3,"belongCmd":"ADD,DEL,SHW,QRY"},
        {"tableId":702,"tableName":"MO_CELLCFG","paramNum":6,"allParamNum":6,"belongCmd":"ADD,DEL,QRY"},
        {"tableId":703,"tableName":"MO_MACFG","paramNum":14,"allParamNum":14,"belongCmd":"ADD,DEL,MOD,QRY"},
        {"tableId":704,"tableName":"MO_CSACFG","paramNum":5,"allParamNum":5,"belongCmd":"ADD,DEL,MOD,QRY"},
        {"tableId":799,"tableName":"MO_MCEDBINFO","paramNum":3,"allParamNum":3,"belongCmd":"QRY"}
        ],
    "mapNumber":6,

    "allParaInfo":[
        {"moId":700, "colId":1, "colName":"MceId", "colDefType":5, "colDefLen":2, "isDynamic":0, "isVisible":1},
        {"moId":700, "colId":2, "colName":"MceName", "colDefType":8, "colDefLen":151, "isDynamic":0, "isVisible":1},
        {"moId":700, "colId":3, "colName":"TimeToWait", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":700, "colId":4, "colName":"MceMcc", "colDefType":8, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":700, "colId":5, "colName":"MceMnc", "colDefType":8, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":700, "colId":6, "colName":"SyncPeriod", "colDefType":5, "colDefLen":2, "isDynamic":0, "isVisible":1},

        {"moId":701, "colId":1, "colName":"M3InterfaceId", "colDefType":4, "colDefLen":1, "isDynamic":0, "isVisible":1},
        {"moId":701, "colId":2, "colName":"M3AssocId", "colDefType":4, "colDefLen":1, "isDynamic":0, "isVisible":1},
        {"moId":701, "colId":3, "colName":"M3State", "colDefType":9, "colDefLen":4, "isDynamic":1, "isVisible":1},

        {"moId":702, "colId":1, "colName":"CellIndex", "colDefType":6, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":702, "colId":2, "colName":"mcc", "colDefType":8, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":702, "colId":3, "colName":"mnc", "colDefType":8, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":702, "colId":4, "colName":"eNodebID", "colDefType":6, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":702, "colId":5, "colName":"CellId", "colDefType":4, "colDefLen":1, "isDynamic":0, "isVisible":1},
        {"moId":702, "colId":6, "colName":"CellReservedForOperatorUse", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},

        {"moId":703, "colId":1, "colName":"MaId", "colDefType":6, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":2, "colName":"pdcchlength", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":3, "colName":"McchRepetitionPeriod", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":4, "colName":"McchModificationPeriod", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":6, "colName":"FreqBandIndicator", "colDefType":4, "colDefLen":1, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":7, "colName":"UlEarfcn", "colDefType":5, "colDefLen":2, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":8, "colName":"DlEarfcn", "colDefType":5, "colDefLen":2, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":9, "colName":"ULBandwidth", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":10, "colName":"DLBandwidth", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":11, "colName":"SAI", "colDefType":6, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":12, "colName":"CellIndexArray", "colDefType":19, "colDefLen":255, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":13, "colName":"CommonSfAllocPeriod", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":14, "colName":"MaSinr", "colDefType":1, "colDefLen":2, "isDynamic":0, "isVisible":1},
        {"moId":703, "colId":15, "colName":"CsaIndexArray", "colDefType":19, "colDefLen":17, "isDynamic":0, "isVisible":1},

        {"moId":704, "colId":1, "colName":"CsaIndex", "colDefType":6, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":704, "colId":2, "colName":"RadioFrameAllocationPeriod", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":704, "colId":3, "colName":"RadioFrameAllocationOffset", "colDefType":4, "colDefLen":1, "isDynamic":0, "isVisible":1},
        {"moId":704, "colId":4, "colName":"SubframeAllocationMode", "colDefType":9, "colDefLen":4, "isDynamic":0, "isVisible":1},
        {"moId":704, "colId":5, "colName":"SubframeAllocation", "colDefType":8, "colDefLen":7, "isDynamic":0, "isVisible":1},

        {"moId":799, "colId":1, "colName":"MCEDBVersion", "colDefType":8, "colDefLen":255, "isDynamic":0, "isVisible":1},
        {"moId":799, "colId":2, "colName":"MCEDBDate", "colDefType":16, "colDefLen":11, "isDynamic":0, "isVisible":1},
        {"moId":799, "colId":3, "colName":"MCEDBTime", "colDefType":12, "colDefLen":13, "isDynamic":0, "isVisible":1}
        ],
    "allParaNumber":37,

    "allParaKeyInfo":[
        {"moId":700, "keyLength":0, "key":{"index0":0, "index1":0, "index2":0, "index3":0, "index4":0, "index5":0, "index6":0, "index7":0, "index8":0, "index9":0}, "keyStructLen":0, "staticStructLen":167, "dynamicStructLen":0, "moCat":27},
        {"moId":701, "keyLength":1, "key":{"index0":1, "index1":0, "index2":0, "index3":0, "index4":0, "index5":0, "index6":0, "index7":0, "index8":0, "index9":0}, "keyStructLen":1, "staticStructLen":2, "dynamicStructLen":5, "moCat":27},
        {"moId":702, "keyLength":1, "key":{"index0":1, "index1":0, "index2":0, "index3":0, "index4":0, "index5":0, "index6":0, "index7":0, "index8":0, "index9":0}, "keyStructLen":4, "staticStructLen":21, "dynamicStructLen":4, "moCat":27},
        {"moId":703, "keyLength":1, "key":{"index0":1, "index1":0, "index2":0, "index3":0, "index4":0, "index5":0, "index6":0, "index7":0, "index8":0, "index9":0}, "keyStructLen":4, "staticStructLen":311, "dynamicStructLen":4, "moCat":27},
        {"moId":704, "keyLength":1, "key":{"index0":1, "index1":0, "index2":0, "index3":0, "index4":0, "index5":0, "index6":0, "index7":0, "index8":0, "index9":0}, "keyStructLen":4, "staticStructLen":20, "dynamicStructLen":4, "moCat":27},
        {"moId":799, "keyLength":0, "key":{"index0":0, "index1":0, "index2":0, "index3":0, "index4":0, "index5":0, "index6":0, "index7":0, "index8":0, "index9":0}, "keyStructLen":0, "staticStructLen":279, "dynamicStructLen":0, "moCat":27}
        ],
    "allParaKeyNumber":6
}
@endjson
