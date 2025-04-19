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
