# bash手册
搜索: Bash Manual
[bash手册之变量扩展](https://www.gnu.org/software/bash/manual/bash.html)

# 变量替换
| Variable                        | Description                                                      |
| ------------------------------- | ---------------------------------------------------------------- |
| `${parameter:-defaultValue}`    | Get default shell variables value                                |
| `${parameter:=defaultValue}`    | Set default shell variables value                                |
| `${parameter:?"Error Message"}` | Display an error message if parameter is not set                 |
| `${#var}`                       | Find the length of the string                                    |
| `${var%pattern}`                | Remove from shortest rear (end) pattern                          |
| `${var%%pattern}`               | Remove from longest rear (end) pattern                           |
| `${var:num1:num2}`              | Substring                                                        |
| `${var#pattern}`                | Remove from shortest front pattern                               |
| `${var##pattern}`               | Remove from longest front pattern                                |
| `${var/pattern/string}`         | Find and replace (only replace first occurrence)                 |
| `${var//pattern/string}`        | Find and replace all occurrences                                 |
| `${!prefix*}`                   | Expands to the names of variables whose names begin with prefix. |
| `${var,}` `${var,pattern}`      | Convert first character to lowercase.                            |
| `${var,,}` `${var,,pattern}`    | Convert all characters to lowercase.                             |
| `${var^}` `${var^pattern}`      | Convert first character to uppercase.                            |
| `${var^^}` `${var^^pattern}`    | Convert all character to uppercase.                              |

## 有没有冒号的区别
* `${parameter:-defaultValue}`  
有冒号, 表示如果parameter不存在, 或者为空, 则返回defaultValue
* `${parameter-defaultValue}`  
没有冒号`:`, 表示如果parameter不存在, 则返回defaultValue

> When not performing substring expansion, using the form described below (e.g., ‘:-’), Bash tests for a parameter that is unset or null. Omitting the colon results in a test only for a parameter that is unset. Put another way, if the colon is included, the operator tests for both parameter’s existence and that its value is not null; if the colon is omitted, the operator tests only for existence.

# 用冒号初始化变量
比如:  
```shell
: "${LANG_CXX:=true}"
: "${LANG_D:=true}"
: "${LANG_OBJC:=true}"
: "${LANG_GO:=true}"
: "${LANG_FORTRAN:=true}"
: "${LANG_ADA:=true}"
: "${LANG_JIT:=true}"
```
* 冒号是个空命令, 什么也不干, 永远返回0
* 但冒号命令会展开参数, 最后的效果是给变量赋默认值
* 加冒号的目的是避免shell把默认值当作命令执行

# export有什么用? 子进程不是继承父进程的环境变量吗?
回答:
* export用于把当前的shell变量export给子进程.
* shell变量并不一定是env变量. 只有export的变量才是环境变量, 没有export的变量压根就不是环境变量, 当然不能被子shell继承.

参考: https://unix.stackexchange.com/questions/130985/if-processes-inherit-the-parents-environment-why-do-we-need-export
