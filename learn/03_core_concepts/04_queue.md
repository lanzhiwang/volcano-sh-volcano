# Queue

default queue

volcano 启动后, 会默认创建名为 default 的 queue. 后续下发的 job, 若未指定 queue, 默认属于 default queue

root queue

volcano 启动后, 同样会默认创建名为 root 的 queue, 该 queue 为开启层级队列功能时使用, 作为所有队列的根队列, default queue 为 root queue 的子队列
