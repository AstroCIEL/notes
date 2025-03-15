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

## `zip`

```bash
zip -r dir.zip dir
unzip filename.zip -d filedir
```

## `tar`

```bash
tar -zcvf [file_name].tar.gz [dir] #打包成.gz文件
tar -zxvf [file_name].tar.gz -C [dir] #解压.gz文件到指定目录
```

## `gzip`

```bash
gzip [file_name] #压缩文件，但是会删除原文件
gzip -c [file_name] > [file_name].gz #压缩文件，保留原文件

gzip -d [file_name].gz #解压文件
```

## check storage usage using `du`

```bash
du -sh [dir] #查看dir的总大小

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
