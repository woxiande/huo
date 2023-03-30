# centos 启用系统日志审计

`vi  /etc/profile`

末尾插入

`` export L_USER=`who am i | awk '{ print $1 }'`
export L_IP=`who am i | awk '{ print $5 }' | sed 's/[()]//g'`
LOGINMSG=[$(date "+%F %T")][L_USER:${L_USER}][${L_IP}][E_USER:$(whoami)]***login***
logger -p local1.info "$LOGINMSG"
export HISTTIMEFORMAT="[%F %T] "
export PROMPT_COMMAND='{ CMD_TIME=$(history 1|{ read n date time cmd; echo "$date $time"; }); CMD=$(history 1|{ read n date time cmd; echo "$cmd"; });logger -p local1.info "${CMD_TIME}[L_USER:${L_USER}][${L_IP}][E_USER:$(whoami)]$CMD"; }' ``

运行

`source /etc/profile`

查看日志文件

`cat /var/log/messages`

以上对用户的操作详细记录
