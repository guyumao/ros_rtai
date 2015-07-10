# ros_rtai
1.创建a.sh:
#!/bin/bash
#开启必要的rtai module 
sync
insmod /usr/realtime/modules/rtai_hal.ko
insmod /usr/realtime/modules/rtai_sched.ko
echo "a"
sync
2.创建b.sh:
#!/bin/bash
#关闭rtai module
sync
rmmod /usr/realtime/modules/rtai_sched.ko
rmmod /usr/realtime/modules/rtai_hal.ko
echo "b"
sync
3.创建main.cpp：
#include <iostream>
#include <cstdlib>
#include <csignal>
#include <rtai_lxrt.h>
using namespace std;
static int end=1;
#信号量在rtai下的支持很不理想，目前没有解决
void endHandler(int sig){
        end=0;
        exit(sig);
}
int main(void){
        RT_TASK *task=0;
        int period=0;
	#接受信号量
        signal(SIGKILL,endHandler);
        signal(SIGTERM,endHandler);
        signal(SIGALRM,endHandler);
        if(!(task=rt_task_init_schmod(nam2num("TEST"),0,0,0,SCHED_FIFO,0x0f))){
                cout <<"Can't initial the task" <<endl;
                exit(1);
        }#如果编译报nam2num的error 将rt_task_init_schmod的第一个参数改成1～1000..的任意整数即可，应该是连接问题，可能要重新该rtai编译选项。
        mlockall(MCL_CURRENT | MCL_FUTURE);
        period=start_rt_timer(nano2count(10000));
        rt_make_hard_real_time();
        rt_task_make_periodic(task,rt_get_time()+period*10,period*100);
        while(end){
                rt_printk("Hello World!\n");
                rt_task_wait_period();
        }
        rt_make_soft_real_time();
        stop_rt_timer();
        rt_task_delete(task);
        cout <<"End of the Application!" <<endl;
        return 0;
}

4.makefile文件：
CC=g++
NAME=main
CFLAGS= -I/usr/realtime/include -o
DFLAGS= -L/usr/realtime/lib -llxrt -lpthread
TARGET=$(NAME)
SOURCE=$(NAME).cpp
all:
        $(CC) $(SOURCE) $(CFLAGS) $(TARGET) $(DFLAGS)
clean:
        rm $(TARGET)

5.运行过程：
sudo ./a.sh
make
sudo chmod +x main
./main
#如果系统没挂的話：可能性不大
make clean
./b.sh
