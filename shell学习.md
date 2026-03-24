### 1. `$()` —— 命令替换 (Command Substitution)

**功能**：运行括号里的命令，并将命令的**输出结果**（标准输出）作为字符串返回。  
**等价写法**：它等价于旧式的反引号  `command` ，但  `$ ()` 更推荐，因为支持嵌套且易读。

#### 示例：

```bash
# 1. 获取当前日期
TODAY=$(date +%Y-%m-%d)
echo "今天是 $TODAY" 
# 输出：今天是 2026-03-20 (假设今天是这天)

# 2. 获取文件行数
LINE_COUNT=$(wc -l < docker-compose.yml)
echo "文件有 $LINE_COUNT 行"

# 3. 嵌套使用 (这是反引号很难做到的)
# 先列出文件，再统计数量
COUNT=$(ls $(pwd) | wc -l)
```

**关键点**：括号里必须是**可执行的命令**（如 `date`, `ls`, `docker`, `grep` 等）。如果你放一个变量名进去，Shell 会试图把这个变量名当作命令去运行，通常会报错 `command not found`。

---



### 2.  `$ {}` —— 变量扩展 (Parameter Expansion)

**功能**：引用变量的值，并可以对这些值进行**高级处理**（如截取字符串、设置默认值、判断是否存在等）。  
**基础用法**： `$ {VAR}` 和  `$ VAR` 通常是一样的，但在复杂场景下必须用  `$ {}`。

####  常见的高级用法（你之前的脚本里用了很多）：

| 语法                 | 含义            | 示例                                     | 结果            |
| ------------------ | ------------- | -------------------------------------- | ------------- |
| `$ {VAR}`          | 基础引用          | `NAME="Hi"; echo $ {NAME}`             | `Hi`          |
| `$ {VAR:-default}` | 默认值 (最常用)     | `echo $ {UNSET_VAR:-"app"}`            | `app` (若变量为空) |
| `$ {#VAR}`         | 获取长度          | `STR="hello"; echo $ {#STR}`           | `5`           |
| `$ {VAR:0:3}`      | 截取子串          | `STR="hello"; echo $ {STR:0:3}`        | `hel`         |
| `$ {VAR%suffix}`   | 删除结尾匹配        | `FILE="img.png"; echo $ {FILE%.png}`   | `img`         |
| `$ {VAR#prefix}`   | 删除开头匹配        | `PATH="/usr/bin"; echo $ {PATH#/usr/}` | `bin`         |
| `$ {VAR^^}`        | 转大写 (Bash 4+) | `str="hi"; echo $ {str^^}`             | `HI`          |
