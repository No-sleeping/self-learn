```sh
# 先拉镜像
docker pull mellanox/tcpdump-rdma
# 通过docker起
docker run -it -v /dev/infiniband:/dev/infiniband -v /tmp/traces:/tmp/traces --net=host --privileged mellanox/tcpdump-rdma /bin/bash
# 进docker 之后再执行以下
tcpdump -i mlx5_1 -w ****.cap -c 2000
```

额外 ：

【镜像打包再使用】

```sh
# 先查看docker下的镜像
1、docker images 
2、docker save -o tcpdump-rdma.tar mellanox/tcpdump-rdma:latest
# 下载到别的机器后，去目标机器
3、docker load -i tcpdump-rdma.tar
4、docker images 
5、docker run ** 
```

<br/>

参考：

https://hub.docker.com/r/mellanox/tcpdump-rdma

https://girondi.net/post/tcpdump_to_wireshark/
