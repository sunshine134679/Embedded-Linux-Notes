# 第19讲 shell脚本条件判断、函数和循环

> 来源：正点原子官方笔记整理

## 一、Shell 脚本条件判断

Shell 脚本支持条件判断。虽然可以通过 `&&` 和 `||` 实现简单判断，但复杂场景更适合使用 `if` 或 `case`。

### 1.1 `if then`

```bash
if 条件判断; then
	# 判断成立要做的事情
fi
```

### 1.2 `if then else`

```bash
if 条件判断; then
	# 条件判断成立要做的事情
else
	# 条件判断不成立要做的事情
fi
```

### 1.3 `if elif else`

```bash
if 条件判断; then
	# 条件判断成立要做的事情
elif [ 条件判断 ]; then
	# elif 条件成立要做的事情
else
	# 条件判断不成立要做的事情
fi
```

### 1.4 `case`

```bash
case $变量 in
	"第1个变量内容")
		# 程序段
		;;
	"第2个变量内容")
		# 程序段
		;;
	"第n个变量内容")
		# 程序段
		;;
esac
```

`;;` 表示该程序块结束。

## 二、Shell 脚本函数

函数写法：

```bash
function fname() {
	# 函数代码段
}
```

## 三、Shell 循环

### 3.1 `while do done`

`while` 表示条件成立时一直循环，直到条件不成立。

```bash
while [ 条件 ]
do
	# 循环代码段
done
```

### 3.2 `until do done`

`until` 表示条件不成立时循环，条件成立后停止。

```bash
until [ 条件 ]
do
	# 循环代码段
done
```

### 3.3 `for in`

`for in` 适合遍历一组值。

```bash
for var in con1 con2 con3
do
	# 循环代码段
done
```

### 3.4 数值 `for` 循环

```bash
for ((初始值; 限制值; 执行步长))
do
	# 循环代码段
done
```
