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
gzip [file_name] #压缩文件
gzip -d [file_name].gz #解压文件
```

## check storage usage using `du`

```bash
du -sh [dir] #查看dir的总大小

du -h --max-depth=1 [dir] #查看dir下各个子目录的大小
```
