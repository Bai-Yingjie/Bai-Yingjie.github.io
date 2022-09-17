- [mkimage](#mkimage)
- [shell devmem脚本](#shell-devmem脚本)
- [shell判断是否包含字串](#shell判断是否包含字串)
- [awk处理表格, 以mkimage为例](#awk处理表格-以mkimage为例)
- [记录日志](#记录日志)
- [等待一个文件](#等待一个文件)
- [发送SIGHUP到pid并等待进程终止](#发送sighup到pid并等待进程终止)
- [pid到进程名](#pid到进程名)
- [mac地址转换到字符串](#mac地址转换到字符串)
- [大小写转换](#大小写转换)
- [去掉前后空格](#去掉前后空格)
- [加0x前缀, 负数时加-0x](#加0x前缀-负数时加-0x)
- [两个16进制数相加](#两个16进制数相加)
- [查看进程是否还在](#查看进程是否还在)
- [网络接口是否存在](#网络接口是否存在)
- [解析命令参数](#解析命令参数)

# mkimage
```sh
#!/bin/sh
  
get_value_by_key()
{
    local str="$1"
    local key=$2
    local value=""
  
    if [ "${str/$key=}" != "$str" ]; then
        value=${str##*$key=}
        value=${value%% *}
        value=${value%%,*}
    fi
    echo $value
}
  
generate_ubi_cfg()
{
#ubi config
cat > $ubifs_cfg << UBI_CFG
[ubifs0]
mode=ubi
image=$ubifs_img1
vol_id=0
vol_size=$leb_size
vol_type=dynamic
vol_name=meta
vol_alignment=1
  
[ubifs1]
mode=ubi
image=$ubifs_img2
vol_id=1
vol_size=$fssize
vol_type=dynamic
vol_name=data
vol_alignment=1
vol_flags=autoresize
UBI_CFG
}
  
make_ubifs_image()
{
    local str="$1"
    local ubi_img=$2
  
    local mk_ubifs=$buildroot_dir/output/host/usr/sbin/mkfs.ubifs
    local mk_ubinize=$buildroot_dir/output/host/usr/sbin/ubinize
  
    #generate mkfs.ubifs if necessary
    if [ ! -f $mk_ubifs ]; then
        echo "Generating mkfs tool..."
        local cwd=`pwd`
        cd $buildroot_dir
        make host-mtd
        cd $cwd
        echo "done"
    fi
  
    [ ! -f $mk_ubifs ] && echo "Make fs tool failed" && return 1
  
    local fsroot=`get_value_by_key "$str" fsroot`
    local fssize=`get_value_by_key "$str" fssize`
  
    local ubifs_img1=/tmp/tmp_img_file1
    local ubifs_img2=/tmp/tmp_img_file2
    local ubifs_cfg=/tmp/tmp_img_cfg
  
    #make meta
    dd if=/dev/zero of=$ubifs_img1 bs=$leb_size count=1
    dd if=$volume_config_file of=$ubifs_img1 conv=notrunc || return 1
    #make data
    $mk_ubifs -r $fsroot -m $min_io_size -e $leb_size -c $max_leb_cnt -o $ubifs_img2 || return 1
  
    #generate config file
    generate_ubi_cfg
  
    #output the image
    $mk_ubinize -o $ubi_img -m $min_io_size -p 128KiB $ubifs_cfg || return 1
  
    return 0
}
  
add2image()
{
    local str="$1"
    local outfile=$2
    local offset=${str%%:*}
    local file=`get_value_by_key "$str" file`
    local size=`get_value_by_key "$str" size`
    local content=`get_value_by_key "$str" content`
    local fstype=`get_value_by_key "$str" fstype`
  
    [ -z "$offset" ] && return 1
    #from hex to dec
    offset=$(($offset))
  
    if [ -n "$content" ]; then
        printf "$content\0" > /tmp/tmp_file
        dd if=/tmp/tmp_file of=$outfile bs=1 conv=notrunc seek=$offset || return 1
    fi
  
    if [ -n "$file" ]; then
        if [ -n "$size" ]; then
            dd if=$file of=$outfile bs=1 conv=notrunc seek=$offset count=$size || return 1
        else
            dd if=$file of=$outfile bs=128K conv=notrunc seek=$((offset/128/1024)) || return 1
        fi  
    fi 
  
    if [ "$fstype" = "ubifs" ]; then
        local img_out=ubi.img
        rm -f $img_out
        make_ubifs_image "$str" $img_out || return 1
        dd if=$img_out of=$outfile bs=128k conv=notrunc seek=$((offset/128/1024)) || return 1
    fi
  
    return 0
}
  
#load config
. *_image.cfg
  
#generate the whole empty image
dd if=/dev/zero bs=$flash_size count=1 | tr '\000' '\377' > $outfile
  
echo "$flash_map" | while read line
do
    if [ -n "$line" ]; then
        add2image "$line" $outfile || exit 1
    fi
done
  
if [ "$?" = "0" ]; then
    #print map
    echo "====SUCCESS===="
    echo "$flash_map"
else
    echo "====FAILURE===="
    echo see *_image.cfg
fi
```


# shell devmem脚本
```sh
#!/bin/sh
 
while true;
do
        devmem 0x100003003005C 32 0x1000000
        devmem 0x100003013005C 32 0x1000000
        devmem 0x100003043005C 32 0x1000000
        devmem 0x100003053005C 32 0x1000000
        devmem 0x100003083005C 32 0x1000000
        devmem 0x100003093005C 32 0x1000000
        devmem 0x1000030c3005C 32 0x1000000
        devmem 0x1000030d3005C 32 0x1000000
 
        devmem 0x100005003005C 32 0x1000000
        devmem 0x100005013005C 32 0x1000000
        devmem 0x100005043005C 32 0x1000000
        devmem 0x100005053005C 32 0x1000000
        devmem 0x100005083005C 32 0x1000000
        devmem 0x100005093005C 32 0x1000000
        devmem 0x1000050c3005C 32 0x1000000
        devmem 0x1000050d3005C 32 0x1000000
done
```


# shell判断是否包含字串
注意:`[[ "$line2" != "$line1"* ]]`中
* 双方括号不能少
* 通配符*不能在引号内
* 含通配符的字符串不能是左操作符
```sh
common_path_2line() { #(line1, line2)
    local line1=$1
    local line2=$2
    while [[ "$line2" != "$line1"* ]];do
        line1=`dirname $line1`
    done
    echo $line1
}
```

# awk处理表格, 以mkimage为例
fpxt-b_image.conf:
```sh
#----------------------------------------------------------------------------------
# start : type : source : descption
#----------------------------------------------------------------------------------
0x00000000 : file : u-boot.bin : bps uboot
0x00120000 : empty : : bps flags sector, leave empty
0x00140000 : string : committed : pkg A status, put committed
0x00160000 : string : uncommitted : pkg B status, put uncommitted
0x00180000 : file : u-boot.itb : uboot package A fit image
0x002C0000 : empty : : uboot package B fit image, leave empty
0x00400000 : file : vmlinux.itb : linux fit image(vmlinux and rootfs)
0x01FE0000 : empty : : uboot env variables, leave empty
```


处理上面文件的脚本:
```sh
create_blank_image() #(size)
{
    dd if=/dev/zero bs=$1 count=1 | tr '\000' '\377' > $image_file
}
 
fill_data_from_file() #(start,file)
{
    local start=$1
    local file=$2
 
    [ ! -f $file ] && echo "#### **ERROR** No such file: $file" && exit 1
    dd if=$file of=$image_file bs=128K conv=notrunc seek=$((start/128/1024))
}
 
fill_data_from_string() #(start,string)
{
    local start=$1
    local string=$2
    local tmpfile=`mktemp`
 
    printf "$string\0" > $tmpfile
    fill_data_from_file $start $tmpfile
    rm -rf $tmpfile
}


awk_wrapper() #(start,type,source)
{
    local start=$1
    local type=$2
    local source=$3
 
    case "$type" in
        file)
            fill_data_from_file $start $source_dir/$source
            ;;
        string)
            fill_data_from_string $start $source
            ;;
        empty)
            ;;
        *)
            ;;
    esac
}

# export for awk sub-shell used
export -f awk_wrapper fill_data_from_file fill_data_from_string
export source_dir image_file

echo "#### fill image data"
cat "$image_conf" | sed '/^#/d'| sed '/^$/d' | awk -F: '{
    print "####" $4;
    ret=system("awk_wrapper "$1" "$2" "$3" ");
    if(ret!=0) {
        exit ret
    }
}'
```

# 记录日志
```sh
SYSLOG_NAME=`basename $0`                                       
 
log()                                                           
{                                                              
    if [ -n "$SYSLOG_NAME" ]                                   
    then                                                       
        LOGGER_ARGS="-t $SYSLOG_NAME"                          
    fi                                                        
 
    if [ -n "$SYSLOG_PRIORITY" ]                              
    then                                                      
        LOGGER_ARGS="$LOGGER_ARGS -p $SYSLOG_PRIORITY"        
    fi                                                        
 
    logger -s $LOGGER_ARGS "$@"                               
}                                                             
 
debug()                                                       
{                                                             
    if config_flag_enabled verbose; then                      
        SYSLOG_PRIORITY=user.debug log "$@"                   
    fi                                                        
}                                                             
 
info()                                                                     
{                                                                          
    if config_flag_enabled verbose; then                                   
        SYSLOG_PRIORITY=user.info log "$@"                                 
    fi                                                                     
}                                                                          
 
warning()                                                                  
{                                                                          
    SYSLOG_PRIORITY=user.warn log "$@"                                     
}                                                                          
 
error()                                                                    
{                                                                          
    SYSLOG_PRIORITY=user.error log "$@"                                    
}
```


# 等待一个文件
```sh
wait_for_file()                                                
{                                                              
    local retval=0                                             
    local file=$1                                              
    local cnt=0                                                
    local cntmax                                               
    if [ -z "$2" ]; then                                       
        cntmax=20                                              
    else                                                       
        cntmax=$2                                              
    fi                                                         
    echo -n Waiting for $file...                               
    while [ ! -e $file -a $cnt -le $cntmax ]; do               
        usleep 100000 #100 ms                                  
        let 'cnt++'                                            
    done                                                       
    if [ -e $file ]; then                                      
        echo " ok (waited $cnt * 100 ms)"                                   
    else                                                                    
        echo " timeout ($cntmax * 100 ms)! System behavior is unspecified..."
        retval=1                                                            
    fi                                                                      
    return $retval                                                          
}    
```

# 发送SIGHUP到pid并等待进程终止
```sh
wait_app_done_pid()
{
    local pid=$1
    # if reading the cmdline in the application's proc directory
    # fails, the application most has most likely already stopped
    local app
    if ! app=$(cat /proc/$pid/task/$pid/stat 2>/dev/null); then
        return 0
    fi

    app=${app#*\(}
    app=${app%%\)*}

    local cnt=0
    local cntmax
    if [ -z "$2" ]; then
        cntmax=60
    else
        cntmax=$2
    fi

    echo -n "Waiting up to $cntmax seconds for $app (PID $pid) to go down..."
    while kill -s 0 "$pid" 2>/dev/null && [ $cnt -le $cntmax ]; do
        sleep 1
        let 'cnt++'
    done
    if [ $cnt -le $cntmax ]; then
          echo "ok, $app (PID $pid) finished in $cnt seconds."
          return 0
    else
          echo "darn, $app (PID $pid) did not finish within $cntmax seconds.. oh well.."
          return 1
    fi
}
```


# pid到进程名
```sh
pid_to_procname()
{
    local pid=$1
    local app=$(cat /proc/$pid/task/$pid/stat 2>/dev/null)
    app=${app#*(}
    app=${app%%)*}
    echo $app
}
```

# mac地址转换到字符串
```sh
mac_hex_to_string()
{
    local hex=$1
    echo $hex | sed -e 's/../&:/g' -e 's/:$//'
}
```

# 大小写转换
```sh
tolower()
{
    local str=$1
    echo $str | tr '[:upper:]' '[:lower:]'
}

toupper()
{
    local str=$1
    echo $str | tr '[:lower:]' '[:upper:]'
}
```

# 去掉前后空格
```sh
trim()
{
    local str=$1
    echo $str | sed -e 's/^ *//g' -e 's/ *$//g'
}
```

# 加0x前缀, 负数时加-0x
```sh
hex_prefix()
{
    # add 0x as prefix if not yet present (but take into account negative numbers)
    echo $1 | sed -e 's/0x//I' -e 's/^\(-\?\)/\10x/'
}
```

# 两个16进制数相加
```sh
addhex()
{
    # sum two hex numbers
    # input can but doesn't have to contain 0x prefix, output will be without 0x
    local res
    let "res = $(hex_prefix $1) + $(hex_prefix $2)"
    printf '%x\n' $res
}
```

# 查看进程是否还在
```sh
isalive()
{
    # Simply return the return code of pidof (0=alive, 1=dead)
    pidof "$@" > /dev/null
}
```

# 网络接口是否存在
```sh
interface_exists()
{
    local iface=$1
    [ -e /sys/class/net/$iface ]
}
```

# 解析命令参数
支持key=value格式转换key为变量, 注意until和shift的用法
```sh
# Alternative getopt that accepts input of the form
# "key1=value1 key2=value2" and exposes key1 and key2
# as shell variables.
# Usage: getopt_simple <cmdline>
getopt_simple()
{
    until [ -z "$1" ]
    do
        parameter=${1%%=*} # Extract name.
        value=${1##*=} # Extract value.
        # Verify if parameter starts with a valid alpha-character, or ignore
        case $parameter in
            [A-z]*) eval $parameter=\"$value\" ;;
        esac
        shift # Drop $1 and shift $2 to $1
    done
}

# Expose only the selected key/value pair. This is safer than getopt_simple
# because it avoids overwriting variables of the calling script unexpectedly.
# Usage: getopt_simple_onevar <key> <cmdline>
getopt_simple_onevar()
{
    local key=$1
    shift

    until [ -z "$1" ]
    do
        parameter=${1%%=*} # Extract name.
        value=${1##*=} # Extract value.
        if [ "$parameter" = "$key" ]; then
            eval $parameter=\"$value\"
        fi
        shift # Drop $1 and shift $2 to $1
    done
}
```
