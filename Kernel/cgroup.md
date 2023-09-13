# 1、cpu限制使用

1.1 在 cpu 资源控制器目录中创建子目录

```sh
mkdir /sys/fs/cgroup/cpu/test1/
```

1.2 设置cpu 

```sh
cat test1/cpu.cfs_period_us
100000
cat test1/cpu.cfs_quota_us 
-1   // -1表示不限制
```

1.3 任务列表

```sh
# 当前cgroup下，所有需要限制的进程号
cat test1/tasks
```

1.4 验证

```sh
# 起一个占用1c进程
sha1sum /dev/zero &
[1] 30635

# 当前占用 100% cpu
30635 root      20   0  116612   1200    884 R 100.0  0.0   3:45.97 sha1sum

# pid 放到 task里面
echo 30635 > test1/tasks

# 限制当前cgroup的cpu 使用
# 这里相当于限制了0.5c
echo 50000 > test1/cpu.cfs_quota_us

# 当前占用 50% cpu
30635 root      20   0  116612   1200    884 R 50.0  0.0   5:43.17 sha1sum
```

1.5 清除cgroup

```sh
# pwd
/sys/fs/cgroup
# cgdelete cpu:/test1/
```

<br/>

1.6 讲虚拟机对应的线程号，传入tasks

```sh
ps -ef |grep qemu |awk '{print $2}'|xargs -i ps -T -p {}|grep CPU |awk '{print $2}'| while read line; do echo $line;  done
```

---

https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/setting-cpu-limits-to-applications-using-cgroups-v1_setting-limits-for-applications

https://blog.csdn.net/ccccsy99/article/details/109169006
