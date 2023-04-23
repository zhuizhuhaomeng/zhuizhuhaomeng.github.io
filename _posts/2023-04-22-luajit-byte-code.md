---
layout: post
title: "LuaJIT 字节码学习"
description: "LuaJIT 字节码学习"
date: 2023-04-22
tags: [LuaJIT，bytecode]
---

# 两个立即数相加

## lua 代码

```lua
1	local c
2
3	c = 1 + 2
4	print(c)
```

## luajit 字节码

```lua
$ luajit -bL t.lua:w
-- BYTECODE -- t.lua:0-5
KGC    0    "print"
0001     [1]    KPRI     0   0      ; set c to nil
0002     [3]    KSHORT   0   3      ; set reg 0 to 3
0003     [4]    GGET     1   0      ; "print"
0004     [4]    MOV      3   0      ; move reg 0 to reg 3
0005     [4]    CALL     1   1   2  ; call func on reg 1 with 1 arg at reg 3 whit 0 return val
0006     [4]    RET0     0   1      ; return with 0 return val
```

可见两个常量相加已经被优化了。

# 两个变量相加

## lua代码

```
1	local a
2	local b
3	local c
4
5	a = 1
6	b = 2
7	c = a + b
8	print(c)
```

## luajit 字节码

```lua
$ luajit -bL t.lua
-- BYTECODE -- t.lua:0-9
KGC    0    "print"
0001     [1]    KNIL     0   2      ; set reg 0-2 to nil
0002     [5]    KSHORT   0   1      ; set reg 0 to 1
0003     [6]    KSHORT   1   2      ; sert reg 1 to 2
0004     [7]    ADDVV    2   0   1  ; add reg 0 and reg 1 and store it in reg 2
0005     [8]    GGET     3   0      ; "print"
0006     [8]    MOV      5   2      ; move reg 2 to reg 5
0007     [8]    CALL     3   1   2  ; call "print" fun in reg 3 with 1 arg at reg 5 and 0 return value.
0008     [8]    RET0     0   1      ; return with 0 return value
```

# 比较大小

## lua 代码

```lua
1	local x = 1
2	local y = 2
3	if x < y then
4	  print("t")
5	else
6	  print("f")
7	end
```

## luajit 字节码

```
-- BYTECODE -- t.lua:0-8
KGC    0    "print"
KGC    1    "t"
KGC    2    "f"
0001     [1]    KSHORT   0   1      ; set reg 0 to 1
0002     [2]    KSHORT   1   2      ; set reg 1 to 2
0003     [3]    ISGE     0   1      ; test if reg 0 >= reg 1
0004     [3]    JMP      2 => 0009  ; if true goto 009 and print "f"
0005     [4]    GGET     2   0      ; "print"
0006     [4]    KSTR     4   1      ; "t"
0007     [4]    CALL     2   1   2  ; call "print" with 1 arg at reg 4 and 0 return value
0008     [4]    JMP      2 => 0012
0009     [6] => GGET     2   0      ; "print"
0010     [6]    KSTR     4   2      ; "f"
0011     [6]    CALL     2   1   2
0012     [7] => RET0     0   1
```



# 字符串赋值

## lua 代码

```lua
1	local x = "1"
2	local y = "2"
3	local c = x .. y
4	print(c)
```

## luajit 字节码

```lua
$ luajit -bL t.lua
-- BYTECODE -- t.lua:0-5
KGC    0    "1"
KGC    1    "2"
KGC    2    "print"
0001     [1]    KSTR     0   0      ; "1"
0002     [2]    KSTR     1   1      ; "2"
0003     [3]    MOV      2   0      ; reg[2] = "1"
0004     [3]    MOV      3   1      ; reg[3] = "2"
0005     [3]    CAT      2   2   3  ; reg[2] = CAT(reg[2], reg[3])
0006     [4]    GGET     3   2      ; "print"
0007     [4]    MOV      5   2      ; reg[5] = reg[2]
0008     [4]    CALL     3   1   2  ; print(reg[5])
0009     [4]    RET0     0   1      ; return
```

# 多参数调用

## lua 代码

```lua
1	local x = "1"
2	local y = "2"
3	local c = x .. y
4	print(x, y, c)
```

## lua 字节码

```shell
$ luajit -bL t.lua
-- BYTECODE -- t.lua:0-5
KGC    0    "1"
KGC    1    "2"
KGC    2    "print"
0001     [1]    KSTR     0   0      ; reg[0] = "1"
0002     [2]    KSTR     1   1      ; reg[1] = "2"
0003     [3]    MOV      2   0      ; reg[2] = reg[0]
0004     [3]    MOV      3   1      ; reg[3] = reg[1]
0005     [3]    CAT      2   2   3  ; reg[2] = CAT(reg[2], reg[3])
0006     [4]    GGET     3   2      ; "print"
0007     [4]    MOV      5   0      ; reg[5] = reg[0]
0008     [4]    MOV      6   1      ; reg[6] = reg[1]
0009     [4]    MOV      7   2      ; reg[7] = reg[2]
0010     [4]    CALL     3   1   4  ; print(reg[5], reg[6], reg[7])
0011     [4]    RET0     0   1      ; return
```

# 调用函数的参数是一个函数

## lua 代码

```lua
1	local function mret()
2	    return 10, 20, 30
3	end
4
5	print(mret())
```

## luajit字节码

```
$ luajit -bL t.lua
-- BYTECODE -- t.lua:1-3
0001     [2]    KSHORT   0  10   ;reg[0] = 10
0002     [2]    KSHORT   1  20   ;reg[1] = 20
0003     [2]    KSHORT   2  30   ;reg[2] = 30
0004     [2]    RET      0   4   ;return reg[0], reg[1], reg[2]

-- BYTECODE -- t.lua:0-6
KGC    0    t.lua:1
KGC    1    "print"
0001     [3]    FNEW     0   0      ; t.lua:1
0002     [5]    GGET     1   1      ; "print"
0003     [5]    MOV      3   0      ; reg[3] = mret
0004     [5]    CALL     3   0   1  ; reg[3], ..., reg[3 + MULTRES - 1] = mret()
0005     [5]    CALLM    1   1   0  ; print(reg[1 + 2], ... reg[1 + 2 + MULTRES + 0 - 1])
0006     [5]    RET0     0   1      ; return
```

# 多值函数作为第一个参数

## lua 代码

```lua
1	local function mret()
2	    return 10, 20, 30
3	end
4
5	print(mret(), 123)
```



## luajit 字节码

```lua
[ljl@localhost ~]$ luajit -bL t.lua
-- BYTECODE -- t.lua:1-3
0001     [2]    KSHORT   0  10   ; reg[0] = 10
0002     [2]    KSHORT   1  20   ; reg[1] = 20
0003     [2]    KSHORT   2  30   ; reg[2] = 30
0004     [2]    RET      0   4   ; return reg[0], reg[1], reg[2]

-- BYTECODE -- t.lua:0-6
KGC    0    t.lua:1
KGC    1    "print"
0001     [3]    FNEW     0   0      ; t.lua:1
0002     [5]    GGET     1   1      ; "print"
0003     [5]    MOV      3   0      ; reg[3] = mret
0004     [5]    CALL     3   2   1  ; reg[3] = mret()
0005     [5]    KSHORT   4 123      ; reg[4] = 123
0006     [5]    CALL     1   1   3  ; print(reg[3], reg[4])
0007     [5]    RET0     0   1      ; return
```

# 多值函数作为最后一个参数 CALLM

## ## lua 代码

``` lua
1	local function mret()
2	    return 10, 20, 30
3	end
4
5	print(123,mret())
```

## lua字节码

```lua
-- BYTECODE -- t.lua:1-3
0001     [2]    KSHORT   0  10   ; reg[0] = 10
0002     [2]    KSHORT   1  20   ; reg[1] = 20
0003     [2]    KSHORT   2  30   ; reg[2] = 30
0004     [2]    RET      0   4   ; return reg[0], reg[1], reg[2]

-- BYTECODE -- t.lua:0-6
KGC    0    t.lua:1
KGC    1    "print"
0001     [3]    FNEW     0   0      ; t.lua:1
0002     [5]    GGET     1   1      ; "print"
0003     [5]    KSHORT   3 123      ; reg[3] = 123
0004     [5]    MOV      4   0      ; reg[4] = mret
0005     [5]    CALL     4   0   1  ; reg[4], ..., reg[4 + MULTRES - 1] = mret()
0006     [5]    CALLM    1   1   1  ; print(reg[3], reg[4], ..., reg[3 + MULTRES + 1 - 1])
0007     [5]    RET0     0   1      ; return
```



|OP |A |B |C/D| Description|
|---|---|---|--|--|
|CALLM | base | lit | lit | Call: A, ..., A+B-2 = A(A+2, ..., A+1+C+MULTRES)|

# 函数调用

## 返回值个数多于赋值的个数

### lua 代码

``` lua
1	local function mret()
2	    return 10, 20, 30
3	end
4
5	local a, b = mret()
6	print(123, a, b)
```

### lua字节码

``` lua
-- BYTECODE -- t.lua:1-3
0001     [2]    KSHORT   0  10  ; reg[0] = 10
0002     [2]    KSHORT   1  20  ; reg[1] = 20
0003     [2]    KSHORT   2  30  ; reg[2] = 30
0004     [2]    RET      0   4  ; return reg[0], reg[1], reg[2]

-- BYTECODE -- t.lua:0-7
KGC    0    t.lua:1
KGC    1    "print"
0001     [3]    FNEW     0   0      ; t.lua:1
0002     [5]    MOV      1   0      ; reg[1] = mret
0003     [5]    CALL     1   3   1  ; reg[1], reg[2] = mret()
0004     [6]    GGET     3   1      ; "print"
0005     [6]    KSHORT   5 123      ; reg[5] = 123
0006     [6]    MOV      6   1      ; reg[6] = reg[1]
0007     [6]    MOV      7   2      ; reg[7] = reg[2]
0008     [6]    CALL     3   1   4  ; print(reg[5], reg[6], reg[7])
0009     [6]    RET0     0   1      ; return
```

## 返回值个数少于赋值参数个数

### lua代码

```lua
1	local function mret(a, b, c)
2	    return a + b + c
3	end
4
5	local a = mret(10, 20, 30)
6	print(a)
```

### luajit 字节码

```lua
-- BYTECODE -- t.lua:1-3
0001     [2]    ADDVV    3   0   1  ; reg[3] = reg[0] + reg[1]
0002     [2]    ADDVV    3   3   2  ; reg[3] = reg[3] + reg[2]
0003     [2]    RET1     3   2      ; return reg[3]

-- BYTECODE -- t.lua:0-7
KGC    0    t.lua:1
KGC    1    "print"
0001     [3]    FNEW     0   0      ; t.lua:1
0002     [5]    MOV      1   0      ; reg[1] = mret
0003     [5]    KSHORT   3  10      ; reg[3] = 10
0004     [5]    KSHORT   4  20      ; reg[4] = 20
0005     [5]    KSHORT   5  30      ; ret[5] = 30
0006     [5]    CALL     1   2   4  ; reg[1] = mret(reg[3], reg[4], reg[5])
0007     [6]    GGET     2   1      ; "print"
0008     [6]    MOV      4   1      ; reg[4] = reg[1]
0009     [6]    CALL     2   1   2  ; print(reg[4])
0010     [6]    RET0     0   1      ; return
```

## 返回值个数少于赋值参数个数

### lua代码

```lua
1	local function mret(a, b, c)
2	    return a + b + c
3	end
4
5	local a, b = mret(10, 20, 30)
6	print(a, b)
```

### luajit 字节码

```lua
$ luajit -bL t.lua
-- BYTECODE -- t.lua:1-3
0001     [2]    ADDVV    3   0   1  ; reg[3] = reg[0] + reg[1]
0002     [2]    ADDVV    3   3   2  ; reg[3] = reg[3] + reg[2]
0003     [2]    RET1     3   2      ; return reg[3]

-- BYTECODE -- t.lua:0-7
KGC    0    t.lua:1
KGC    1    "print"
0001     [3]    FNEW     0   0      ; t.lua:1
0002     [5]    MOV      1   0      ; reg[1] = mret
0003     [5]    KSHORT   3  10      ; reg[3] = 10
0004     [5]    KSHORT   4  20      ; reg[4] = 20
0005     [5]    KSHORT   5  30      ; reg[5] = 30
0006     [5]    CALL     1   3   4  ; reg[1], reg[2] = mret(reg[3], reg[4], reg[5])
0007     [6]    GGET     3   1      ; "print"
0008     [6]    MOV      5   1      ; reg[5] = reg[1]
0009     [6]    MOV      6   2      ; reg[6] = reg[2]
0010     [6]    CALL     3   1   3  ; print(reg[5], reg[6])
0011     [6]    RET0     0   1      ; return
```

# 简单 for 循环

## lua代码

```lua
1	local array = {"Google", "Runoob"}
2
3	for key,value in ipairs(array)
4	do
5	   print(key, value)
6	end
```

## 字节码

``` lua
-- BYTECODE -- t.lua:0-7
KGC    0    table
KGC    1    "ipairs"
KGC    2    "print"
0001     [1]    TDUP     0   0      ; reg[0] = tables[0]
0002     [3]    GGET     1   1      ; "ipairs"
0003     [3]    MOV      3   0      ; reg[3] = tables[0]
0004     [3]    CALL     1   4   2  ; reg[1], reg[2], reg[3] = ipairs(reg[3])
0005     [4]    JMP      4 => 0010  ; jump to 0010 if reg[3] ~= nil
0006     [5] => GGET     6   2      ; "print"
0007     [5]    MOV      8   4      ; reg[8] = reg[4]
0008     [5]    MOV      9   5      ; reg[9] = reg[5]
0009     [5]    CALL     6   1   3  ; print(reg[8], reg[9])
0010     [3] => ITERC    4   3   3  ; reg[4], reg[5], reg[6] = reg[1], reg[2], reg[3];
                                    ; reg[4], reg[5] = reg[4](reg[5], reg[6])
0011     [3]    ITERL    4 => 0006  ; goto 0006 if reg[4] ~= nil
0012     [6]    RET0     0   1
```

这里有点解释不通，ITERC下一次执行还是用 reg[1],reg[2],reg[3] 覆盖 reg[4], reg[5], reg[6],
但是reg[2], reg[3] 并没有用 新的 reg[5], reg[6] 覆盖.


|OP |A |B |C/D| Description|
|---|---|---|--|--|
|ITERC | base | lit | lit| A, A+1, A+2 = A-3, A-2, A-1; A, ..., A+B-2 = A(A+1,A+2)|

|OP |A |D | Description|
|---|---|---|--|
|ITERL| base | jump | Iterator 'for' loop |

# for 迭代器

## lua 代码

```lua
 1	local function square(iteratorMaxCount, currentNumber)
 2	   if currentNumber < iteratorMaxCount then
 3	      currentNumber = currentNumber + 1
 4	      return currentNumber, currentNumber * currentNumber
 5	   end
 6	end
 7	
 8	for i, n in square, 3, 0 do
 9	   print(i, n)
10	end
```

## 字节码

```lua
-- BYTECODE -- t.lua:1-6
0001 [t.lua:2]    ISGE     1   0
0002 [t.lua:2]    JMP      2 => 0007  ; if reg[1] >= reg[0] jump to 0007
0003 [t.lua:3]    ADDVN    1   1   0  ; 1
0004 [t.lua:4]    MOV      2   1
0005 [t.lua:4]    MULVV    3   1   1
0006 [t.lua:4]    RET      2   3
0007 [t.lua:6] => RET0     0   1

-- BYTECODE -- t.lua:0-11
KGC    0    t.lua:1
KGC    1    "print"
0001 [t.lua:6]    FNEW     0   0      ; t.lua:1
0002 [t.lua:8]    MOV      1   0      ; reg[1] = square
0003 [t.lua:8]    KSHORT   2   3      ; reg[2] = 3
0004 [t.lua:8]    KSHORT   3   0      ; reg[3] = 0
0005 [t.lua:8]    JMP      4 => 0010  ; jump to 0010 if reg[3] ~= nil
0006 [t.lua:9] => GGET     6   1      ; "print"
0007 [t.lua:9]    MOV      8   4      ; reg[8] = reg[4]
0008 [t.lua:9]    MOV      9   5      ; reg[9] = reg[5]
0009 [t.lua:9]    CALL     6   1   3  ; print(reg[8], reg[9])
0010 [t.lua:8] => ITERC    4   3   3  ; reg[4], reg[5], reg[6] = reg[1], reg[2], reg[3];
                                      ; reg[4], reg[5] = reg[4](reg[5], reg[6])
0011 [t.lua:8]    ITERL    4 => 0006  ; jump to 0006 if reg[4] ~= nil
0012 [t.lua:10]    RET0     0   1
```

# for 循环代码

## lua 代码

```lua
local p = { x = 1, y = 1, [1] = 1 }
for i = 1, 100 do
  p = { x = p.x + i,
        y = p.y - i,
        [1] = p[1] }
end
```

## 字节码

```lua
-- BYTECODE -- t.lua:0-7
KGC    0    table
KGC    1    "x"
KGC    2    table
KGC    3    "y"
0001     [1]    TDUP     0   0
0002     [2]    KSHORT   1   1
0003     [2]    KSHORT   2 100
0004     [2]    KSHORT   3   1
0005     [2]    FORI     1 => 0017  ; jump to 0017 if reg[1] == nil
0006     [3] => TDUP     5   2
0007     [3]    TGETS    6   0   1  ; "x"
0008     [3]    ADDVV    6   6   4  ; reg[6] = reg[6] + reg[4]
0009     [3]    TSETS    6   5   1  ; "x" => reg[5]["x"] = reg[6]
0010     [4]    TGETS    6   0   3  ; "y"
0011     [4]    SUBVV    6   6   4  ; reg[6] = reg[6] + reg[4]
0012     [4]    TSETS    6   5   3  ; "y" => reg[5]["x"] = reg[6]
0013     [5]    TGETB    6   0   1  ; reg[6] = reg[0][1]
0014     [5]    TSETB    6   5   1  ; reg[5][1] = reg[6] 
0015     [5]    MOV      0   5      ; reg[0] = reg[5]
0016     [2]    FORL     1 => 0006  ; if reg[1] <= reg[2]
0017     [6] => RET0     0   1
```
