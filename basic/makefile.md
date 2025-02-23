# Makefile 语法完全指南

## 目录
1. [什么是Makefile？](#什么是makefile)
2. [基本语法结构](#基本语法结构)
3. [变量与赋值](#变量与赋值)
4. [隐式规则与模式匹配](#隐式规则与模式匹配)
5. [函数与条件判断](#函数与条件判断)
6. [高级技巧与最佳实践](#高级技巧与最佳实践)
7. [实战示例](#实战示例)
8. [调试与错误处理](#调试与错误处理)

<a name="什么是makefile"></a> 
## 1. 什么是Makefile？

Makefile 是自动化构建工具 `make` 的配置文件，用于：
- 定义项目构建规则
- 管理源代码到可执行文件的转换
- 自动化编译流程
- 处理文件依赖关系

**基本工作流程**：
```
目标（target） ← 依赖（prerequisites）
    命令（recipe）
```

<a name="基本语法结构"></a>
## 2. 基本语法结构

### 2.1 基本规则
```makefile
target: prerequisites
    recipe
```

**示例**：
```makefile
hello: main.o utils.o
    gcc -o hello main.o utils.o

main.o: main.c
    gcc -c main.c

utils.o: utils.c
    gcc -c utils.c
```

### 2.2 自动变量
| 变量 | 说明                |
|------|--------------------|
| $@   | 当前目标文件名       |
| $<   | 第一个依赖文件       |
| $^   | 所有依赖文件         |
| $?   | 比目标新的依赖文件列表 |

**示例**：
```makefile
%.o: %.c
    gcc -c $< -o $@
```

### 2.3 特殊目标
- `.PHONY`: 声明伪目标（不生成实际文件）
- `.DEFAULT`: 默认目标
- `.IGNORE`: 忽略错误

**示例**：
```makefile
.PHONY: clean
clean:
    rm -f *.o hello
```

<a name="变量与赋值"></a>
## 3. 变量与赋值

### 3.1 变量类型
| 赋值运算符 | 说明                     |
|------------|--------------------------|
| =          | 递归展开（延迟求值）     |
| :=         | 立即展开                 |
| ?=         | 条件赋值（未定义时赋值） |
| +=         | 追加赋值                 |

**示例**：
```makefile
CC := gcc
CFLAGS ?= -O2
SRC = $(wildcard *.c)
OBJ = $(SRC:.c=.o)
```

### 3.2 作用域
- 全局变量：在Makefile顶部定义
- 目标特定变量：
  ```makefile
  target: VAR = value
  ```

<a name="隐式规则与模式匹配"></a>
## 4. 隐式规则与模式匹配

### 4.1 模式规则
```makefile
%.o: %.c
    $(CC) -c $< -o $@
```

### 4.2 静态模式
```makefile
$(OBJ): %.o: %.c
    $(CC) $(CFLAGS) -c $< -o $@
```

### 4.3 常见隐式规则
- 从.c生成.o：自动使用$(CC) -c
- 从.l（lex）生成.c
- 从.y（yacc）生成.c

<a name="函数与条件判断"></a>
## 5. 函数与条件判断

### 5.1 常用函数
```makefile
# 文件搜索
$(wildcard src/*.c)

# 字符串替换
$(patsubst %.c,%.o,$(SRC))

# shell命令
$(shell date +%Y%m%d)
```

### 5.2 条件判断
```makefile
ifeq ($(OS),Windows_NT)
    RM = del /Q
else
    RM = rm -f
endif
```

<a name="高级技巧与最佳实践"></a>
## 6. 高级技巧与最佳实践

### 6.1 文件包含
```makefile
include config.mk
```

### 6.2 并行构建
```bash
make -j4  # 使用4个线程
```

### 6.3 目录创建
```makefile
$(BUILD_DIR):
    mkdir -p $@
```

### 6.4 最佳实践
1. 使用`.PHONY`声明伪目标
2. 避免在recipe中使用绝对路径
3. 保持recipe行缩进为Tab字符
4. 使用`@`前缀隐藏命令回显
5. 添加`help`目标说明用法

<a name="实战示例"></a>
## 7. 实战示例

### C项目模板
```makefile
CC := gcc
CFLAGS := -Wall -O2
SRC_DIR := src
BUILD_DIR := build

SRC := $(wildcard $(SRC_DIR)/*.c)
OBJ := $(patsubst $(SRC_DIR)/%.c,$(BUILD_DIR)/%.o,$(SRC))

.PHONY: all clean

all: $(BUILD_DIR)/program

$(BUILD_DIR)/program: $(OBJ)
    $(CC) $^ -o $@

$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c | $(BUILD_DIR)
    $(CC) $(CFLAGS) -c $< -o $@

$(BUILD_DIR):
    mkdir -p $@

clean:
    rm -rf $(BUILD_DIR)
```

<a name="调试与错误处理"></a>
## 8. 调试与错误处理

### 调试选项
```bash
make --debug             # 显示详细调试信息
make -n                  # 空运行（只显示不执行）
make --print-data-base   # 打印所有规则和变量
```

### 常见错误
1. **Missing separator**：recipe行没有用Tab缩进
2. **No rule to make target**：依赖文件不存在
3. **Command not found**：命令拼写错误或未安装工具

---

> **提示**：使用`make -p`查看内置规则，掌握更多自动化构建技巧
