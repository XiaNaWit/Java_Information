# Bash 与管道

## Bash 特性

- **命令历史**：记录使用过的命令
- **命令与文件补全**：Tab 键
- **命名别名**：如 `ll` 是 `ls -al` 的别名
- **Shell Scripts**：脚本编程
- **通配符**：如 `ls -l /usr/bin/X*` 列出 /usr/bin 下所有 X 开头的文件

## 变量操作

```bash
x=abc                    # 赋值
echo $x                  # 取用变量
echo ${x}

x="lang is $LANG"        # 双引号：特殊字符保留原特性
x='lang is $LANG'        # 单引号：特殊字符为字面值

version=$(uname -r)      # 将指令执行结果赋值给变量
version=`uname -r`

export x                 # 转成环境变量（可在子程序中使用）
```

### declare 声明变量

```bash
declare [-aixr] variable
-a ：数组类型
-i ：整数类型
-x ：环境变量
-r ：只读类型
```

### 数组

```bash
array[1]=a
array[2]=b
echo ${array[1]}
```

## 指令搜索顺序

1. 以绝对/相对路径执行（如 `/bin/ls` 或 `./ls`）
2. 由别名找到指令执行
3. 由 Bash 内置指令执行
4. 按 `$PATH` 路径顺序找到第一个指令

## PATH

环境变量 `PATH` 声明可执行文件的路径，之间用 `:` 分隔。

```
/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/dmtsai/.local/bin:/home/dmtsai/bin
```

## 数据流重定向

| 设备 | 代码 | 运算符 |
|:---:|:---:|:---:|
| 标准输入（stdin） | 0 | `<` 或 `<<` |
| 标准输出（stdout） | 1 | `>` 或 `>>` |
| 标准错误输出（stderr） | 2 | `2>` 或 `2>>` |

`>` 为覆盖，`>>` 为追加。`/dev/null` 为"垃圾箱"。

```bash
find /home -name .bashrc > list 2>&1    # 将 stdout 和 stderr 同时重定向到文件
```

## 管道指令

管道（`|`）将一个命令的标准输出作为另一个命令的标准输入。

```bash
ls -al /etc | less
```

### cut — 提取

```bash
cut -d '分隔符' -f n     # 按分隔符取第 n 个区间
cut -c n1-n2             # 按字符取区间
```

### sort — 排序

```bash
sort [-fbMnrtuk] [file or stdin]
-f ：忽略大小写
-n ：按数字排序
-r ：反向排序
-u ：去重（unique）
-t ：指定分隔符
-k ：指定排序区间
```

### uniq — 去重

```bash
uniq [-ic]
-i ：忽略大小写
-c ：计数
```

### tee — 双向输出重定向

将输出同时写入文件和屏幕。

```bash
tee [-a] file    # -a 追加
```

### tr — 字符转换

```bash
tr [-ds] SET1 ...
-d ：删除 SET1 字符串
```

示例：`last | tr '[a-z]' '[A-Z]'`

### col, expand — 空白字符转换

```bash
col [-xb]                # 将 tab 转为空格
expand [-t] file         # tab 转指定数量空格（默认 8）
```

### join — 合并行

```bash
join [-ti12] file1 file2
-t ：分隔符
-i ：忽略大小写
-1/-2：指定比较字段
```

### paste — 粘贴行

```bash
paste [-d] file1 file2   # -d 指定分隔符，默认 tab
```

### split — 分区

```bash
split [-bl] file PREFIX  # -b 按大小，-l 按行数
```

## 正则表达式

### grep

全局正则搜索并打印。

```bash
grep [-acinv] [--color=auto] 搜寻字符串 filename
-c ：统计匹配行数
-i ：忽略大小写
-n ：输出行号
-v ：反向选择
--color=auto ：关键字加颜色
```

示例：`grep -n 'the' regular_express.txt`

正则示例：`grep -n 'a\{2,5\}' regular_express.txt`

### printf

格式化输出。

```bash
printf '%10s %5i %5i %5i %8.2f \n' $(cat printf.txt)
```

### awk

每次处理一行，以字段为单位，`$n` 为第 n 字段，`$0` 为整行。

```bash
awk '条件类型 1 {动作 1} 条件类型 2 {动作 2} ...' filename
```

**内置变量**：

| 变量 | 说明 |
|:---|:---|
| NF | 每行字段总数 |
| NR | 当前处理行号 |
| FS | 分隔符（默认空格） |

示例：`last -n 5 | awk '{print $1 "\t" $3}'`
