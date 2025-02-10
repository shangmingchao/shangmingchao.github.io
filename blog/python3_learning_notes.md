# Python3 学习笔记

## 数据类型

- `/` 总是返回浮点数，向下取整要用 `//`  
- 幂运算符为 `**`  
- `type()` 不会认为子类是一种父类类型， `isinstance()` 会认为子类是一种父类类型
- 交互模式下最后打印的表达式被赋值给变量 `_` ，可以直接使用它参与后来的运算，但它是只读的，不要主动给它赋值
- 多行字符串可以用三个单引号或三个双引号包裹，可以每行末尾添加 `\` 防止默认自动添加的换行
- 字符串 `+` 表示拼接， `*` 表示重复，两个挨着的字符串字面量不用 `+` 也会自动拼接
- 没有字符类型，只有字符串类型
- 字符串索引（indexing）可以是负的，表示从右开始计数，从 -1 开始，-0 和 0 是一样的
- 为切片赋值甚至可以清空列表：`letters[:] = []`
- 布尔类型的值是 `True` 和 `False`，空类型的值是 `None`  

## 语法

### 循环迭代

for-in 简单迭代：

```python
for i in range(5):
    print(i, end=',' if i < 4 else '\n')
# 0,1,2,3,4
```

带索引的迭代：

```python
fruits = ['apple', 'banana', 'cherry']
for index, fruit in enumerate(fruits):
    print(index, fruit)
# 0 apple
# 1 banana
# 2 cherry
```

### 函数

定义函数时使用 `def` 关键字：

```python
def fib():
    """文档注释"""
```

文档字符串放到函数体的第一行。没有 return 或者 return 没有表达式表明函数返回 `None`


### 输出格式化

```python
num = 3.14

# 使用format()
print("Using format(): {:.2f}".format(num))

# 使用f-string
print(f"Using f-string: {num:.2f}")

# 使用%格式化
print("Using %% formatting: %.2f" % num)

# 使用round()
print("Using round():", round(num, 2))
```

建议使用 f-string  
f-string 中左右中对齐符号为 `<`, `>`, `^`，如 `f"Left aligned: {text:<10}"`。  
f-string 中可以直接调用函数或方法。  
f-string 中可以使用条件表达式：`print(f"Result: {'Pass' if score >= 60 else 'Fail'}")`  
`{num:,.2f}` ：千分位分割且保留两位小数，`{0.25:.2%}` 百分比格式且保留两位小数  

## 参考

- [The Python Tutorial](https://docs.python.org/zh-cn/3.13/tutorial/index.html)
