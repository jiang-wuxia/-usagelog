# 富芮坤-fr30xx系列连接参数详解

```C
typedef struct gap_security_param {
    bool    mitm;  //是否允许中间人认证
    bool    secure_connection;//是否允许加密连接 安全等级 是： level4 否：level3 
    bool    bond;     //是否允许绑定              
    enum gap_sec_mode   rsp_mode;   //  是否手动管理连接加密模式  
    enum gap_oob        oob_used;   //  是否使用OOB数据    
    enum gap_io_cap     io_cap;     //  输入输出能力   
}gap_security_param_t;
```
## 设备做从机
### just work连接模式  
```C
struct gap_security_param smp_param;
    smp_param.mitm = false;
    smp_param.secure_connection = false;
    smp_param.bond = true;
    smp_param.rsp_mode = ENABLE_AUTO_RSP;
    smp_param.oob_used = GAP_OOB_AUTH_DATA_NOT_PRESENT;
    smp_param.io_cap = GAP_IO_CAP_NO_INPUT_NO_OUTPUT;
```
---
***注意***  
---
从机发起安全连接的函数：`gap_security_req`

----
**rsp_mode** = **ENABLE_AUTO_RSP**时 协议栈会自动响应连接请求    
**rsp_mode** = **ENABLE_MANUAL_RSP**时 协议栈不会自动响应连接请求  此时需要手动调用函数响应连接请求。  
要在GAP回调函数里加  
```c
case GAP_EVT_SMP_PAIR_REQ://配对请求事件
    gap_pairing_rsp(event->param.connect.conidx,true);
    break;

case GAP_EVT_SMP_ENCRYPT_REQ://加密请求事件
    printf(" GAP_EVT_SMP_ENCRYPT_REQ \r\n");
        
    gap_bond_info_t out_bond_info;
    memset(&out_bond_info, 0, sizeof(out_bond_info));
    gap_get_bond_info_by_conidx(event->param.connect.conidx,&out_bond_info);
        
    gap_encrypt_rsp(event->param.connect.conidx,out_bond_info.peer_ltk.ltk,true);
    break;  

case GAP_EVT_SMP_ENCRYPT_SUCCESS://加密成功事件
    printf("ENCRYPT_SUCCESS \r\n");
    break;   

case GAP_EVT_SMP_BOND_SUCCESS://绑定成功事件
    printf("bond_SUCCESS \r\n");     
    struct gap_link_param_update_rsp rsp;
    rsp.accept = true;
    rsp.conidx = event->param.link_param_update_req.conidx;
    rsp.ce_len_max = 2;
    rsp.ce_len_min = 2;
    gap_param_update_rsp(&rsp);       
    break;    
```  


---
### PIN码  
设备需要 输入PIN码  

 需要改io_cap参数  
 然后在GAP回调函数里加
```c
case GAP_EVT_SMP_TK_REQ:
    {
        printf("gap_callback: GAP_EVT_SMP_TK_REQ, type:%d\r\n", event->param.tk_req.tk_type);
        
        if((event->param.tk_req.tk_type == GAP_TK_TYPE_DISPLAY) ||(event->param.tk_req.tk_type == GAP_TK_TYPE_ENTRY))               
        {
            uint32_t local_pin_code = 123456; // PIN码
            gap_security_tk_rsp(event->param.nc_req.conidx,local_pin_code,true);//发送PIN值
        } 
    }
        break;

```


  




