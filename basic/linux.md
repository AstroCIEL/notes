# Frequently used commands in linux bash

## find files of specific name(`find`)

```bash
#find files recursively in current dir
find . -name "*.lef"
```

## find contents in a file(`grep`)

```bash
#find 'to_be_found' in file note.txt
grep 'to_be_found' note.txt
#alternative
cat note.txt | grep 'to_be_found'
#here the pipe | transmits the contents to stdin rather than grep
#an argument of grep. grep read from stdin 
```

## find contents in all files of a directory(`grep`)

```bash
# find 'to_be_found' in the current dir recursively
grep -r 'to_be_found' .
```

## find files whose name has certain pattern and delete them

```bash
find . -type f -name "*to_be_found*" -exec rm {} \;
# if you want to be asked when each file is deleted, add -i
find . -type f -name "*to_be_found*" -exec rm -i {} \;
```

## extract and archive with `zip`, `tar`, `gzip`

```bash
# .zip
zip -r dir.zip dir
unzip filename.zip -d filedir

# .tar.gz
tar -zcvf [file_name].tar.gz [dir] #打包成.tar.gz文件
tar -zxvf [file_name].tar.gz -C [dir] #解压.tar.gz文件到指定目录

# .gz
gzip [file_name] #压缩文件，但是会删除原文件
gzip -c [file_name] > [file_name].gz #压缩文件，保留原文件
gzip -d [file_name].gz #解压文件
```

## check storage usage using `du`& `df`

```bash
du -sh [dir] #查看dir的总大小
df -h [dir] #查看目录空余空间

du -h --max-depth=1 [dir] #查看dir下各个子目录的大小
du -h -d 1 [dir] #--max-depth简写-d
```

## manage soft link of directory using `ln`

```bash
# wherever you are
ln -s source/dir/a target/dir/b #创建软链接

# if you're currently in the dir of the target dir
ln -s /source/dir/a b # 在当前目录下创建一个软链接b
ln -s source/dir/a # 缺省目标文件夹名称，在当前目录下创建一个同名软链接a

# if you want to delete the link
rm b #删除软链接b
```

## `tmux`

```bash
sudo apt-get install tmux

tmux new -s <session-name>

tmux detach # or ctrl+b, d

tmux ls # list all sessions

tmux attach -t <session-name> # attach to a session

tmux kill-session -t <session-name>
```

## `conda`常用命令

```bash
conda init

conda create --name <env_name> python=<version>

conda env list

conda env remove --name <env_name>
```

## 权限设置`chmod`

```bash
# 给所有用户添加读和执行权限
chmod -R a+rx .

# 当前用户rwx，其他用户rx
chmod -R u+rwx,g+rx,o+rx . # 或
chmod -R 755 .
```

## 别名`alias`

```bash
# 查看当前所有别名
alias

# 创建别名
alias ll='ls -alF'
unalias ll # 删除别名

# 永久生效
echo "alias ca='conda activate'" >> ~/.bashrc
echo "alias nv='nvidia-smi'" >> ~/.bashrc
echo "alias wnv='watch -n 1 nvidia-smi'" >> ~/.bashrc
echo "alias pi='pip install'" >> ~/.bashrc
echo "alias py='python'" >> ~/.bashrc
echo "alias up='cd ..'" >> ~/.bashrc
echo "alias tnew='tmux new -s'" >> ~/.bashrc
echo "alias tkill='tmux kill-session -t'" >> ~/.bashrc
echo "alias tattach='tmux attach -t'" >> ~/.bashrc
echo "alias pfind='ps aux | grep'" >> ~/.bashrc
source ~/.bashrc     # 立即生效 
```

## nvidia-smi

```bash
# 查看GPU信息
nvidia-smi

# 持续检测GPU信息
nvidia-smi -l 1
# or
watch -n 1 nvidia-smi
```

## docker

安装完docker记得先更新镜像源，否则无法下载任何镜像。

```bash
########image
docker search [镜像名]
docker pull [镜像名]:[标签]  # 默认标签为 latest
docker images  # 查看本地镜像
docker rmi [镜像ID或镜像名]  # 删除镜像

########container
docker run -it --name [容器名] [镜像名] /bin/bash  # 运行容器
docker ps -a  # 查看所有容器
docker start [容器ID或容器名]  # 启动容器
docker stop [容器ID或容器名]  # 停止容器
docker restart [容器ID或容器名]  # 重启容器
docker rm [容器ID或容器名]  # 删除容器
docker exec -it [容器ID或容器名] /bin/bash  # 在容器运行时进入其命令行
docker logs [容器容器ID或容器名名]  # 查看容器日志
docker stats [容器ID或容器名]  # 查看容器资源占用情况
```
