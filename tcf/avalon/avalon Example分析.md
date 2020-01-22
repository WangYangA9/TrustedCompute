# 工作指令
## Worker Order Submit
* python服务负责处理（tcs_work_order_handler.py:139#WorkOrderSubmit）
* 收到请求后，验证各个字段的格式是否合法，是否有重复的任务id等
* 更新lmdb表内，id对应的（lmdb可以用区块链代替）
    * wo-timestamps-时间戳
    * wo-requests-input_json_str
    * wo-scheduled-input_json_str
* workorder_list新增id
* envclave_manager.py的process_work_orders函数在，lmdb的表中轮训，作为消费者从wo-scheduled取出元素
    * 在wo-process中存储该工作指令id
    * 从wo-requests中取出对应id的输入
    * 验证request input的合法性
    * Execute work order request开始调用enclave
    * execute_work_order函数
    * sgx_workorder_request开始处理
    * 将加密数据发给sgx enclave进行处理
    * 调用到work_order_wrap.cpp进行处理
    * 最终在work_order_processor.cpp中进行处理，每次执行一个work_order都建立新的WorkOrderProcessor
    * 根据workload_id选择计算器
    * 返回加密结果
# Example

## Echo 被发送给worker的消息被直接echo

## EEA Token Execution 示范EEA token执行逻辑

## Heart Disease Evaluation 这个而应用展现了输入一个人并计算心脏病概率。客户端支持GUI和命令行形式

## Generic Workload Client 用于与任何Worker进行测试的通用客户端


