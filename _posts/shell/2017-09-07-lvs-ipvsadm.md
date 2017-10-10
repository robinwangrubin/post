---
layout: post
title: lvs部署脚本
categories: Shell
---

```
#!/usr/bin/env bash
#Author: xxxxxxx
#Email: xxxxxxx@gmail.com
#Description: This script is for start stop reload lvs 

WORK_DIR="/root/lvs/"
CMD="/usr/sbin/ipvsadm"
LOCK_FILE="/root/lvs/lvs.lock"

cd $WORK_DIR
case "$1" in
        start)
        if [ -f $LOCK_FILE ];then
                echo -e "\033[32mLVS was Started.\033[0m"
                exit 0
        fi
        echo -e "\033[32mSTARTTING LVS......\033[0m"
        while read line
        do
                $CMD $line
                echo -e "\033[32m$CMD $line\033[0m"
        done < ./lvs_list.txt
        touch $LOCK_FILE
        ;;
        stop)
        if [ -f $LOCK_FILE ];then
                echo -e "\033[32mSTOPPING LVS......\033[0m"
                $CMD -C
                rm -f $LOCK_FILE
        else
                echo -e "\033[32mLVS was Stopped.\033[0m"
                exit 0
        fi
        ;;
        restart)
        echo -e "\033[32mRESTARTTING LVS......\033[0m"
        $0 stop && sleep 1 && $0 start
        ;;
        *)
        echo -e "\033[31mUsage: Pls use sh lvs.sh start|stop|restart|reload\033[0m"
        exit 1
esac
```

----------------

end
