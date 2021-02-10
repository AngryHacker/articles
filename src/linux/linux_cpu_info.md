# linux cpu 信息分析
在 Linux 下如何查看 CPU 信息呢？只要查看 /proc/cpuinfo 文件就好了。

```sh
cat /proc/cpuinfo
```

在我的电脑下得到如下结果：
![linux cpu info](https://raw.githubusercontent.com/AngryHacker/articles/master/img/linux_cpu_info.png)


其中包含了很多信息，比较重要的字段是：

* processor  逻辑处理器的 id，从 0 开始

* physical id 物理处理器的 id，从 0 开始，可以判断电脑中有多少个 CPU

* core id 每个核心的 id

* cpu cores 位于同一个物理处理器中的内核数量，可以看到每个 CPU 有几个物理核

* siblings 位于同一个物理处理器中的逻辑处理器的数量，可以看到一个 CPU 有多少逻辑处理器


写了个简单的 Shell 脚本来判断对应的 CPU 信息：

```sh
#!/bin/bash
echo "CPU 分析菜单:"
echo "1.查看逻辑 cpu 个数"
echo "2.查看物理 cpu 个数"
echo "3.查看每个 cpu 的物理核数"
echo "4.查看每个 cpu 的逻辑处理器数"
echo "5.退出"
read -p "请选择:" input
while [[ $input != '5' ]]
do
    if [[ $input = '1' ]];
    then
        echo -en "\n逻辑处理器共有："
        cat /proc/cpuinfo | grep 'processor' | wc -l
    elif [[ $input = '2' ]];
    then
        echo -en "\n物理处理器共有："
        cat /proc/cpuinfo | grep 'physical id' | sort -u | wc -l
    elif [[ $input = '3' ]];
    then
        echo -en "\n每个 cpu 的物理核数为："
        cat /proc/cpuinfo | grep 'cpu cores' | sort -u | awk -F ':' '{print $2}'
    elif [[ $input = '4' ]];
    then
        echo -en "\n每个 cpu 的逻辑处理器为："
        cat /proc/cpuinfo | grep 'siblings' | sort -u | awk -F ':' '{print $2}'
    else
        echo -e '\n错误输入'
    fi
    echo -e "\nCPU 分析菜单:"
    echo "1.查看逻辑 cpu 个数"
    echo "2.查看物理 cpu 个数"
    echo "3.查看每个 cpu 的物理核数"
    echo "4.查看每个 cpu 的逻辑处理器数"
    echo "5.退出"
    read -p "请选择:" input
done
```