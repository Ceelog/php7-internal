
# 一段 PHP 代码执行之旅

PHP 是一门动态编译执行的高级语言。

从一段代码到获得执行结果，需要经历以下几个过程：

```
源码 -> 词法/语法解析 -> AST抽象语法树 -> 编译为 OPCODE -> Zend虚拟机引擎执行 -> 输出
```

以一段简单的 PHP 代码为例：
```
<?php

function plus($a, $b) {
    return $a + $a;
}

$a = 1;
$b = ++$a;

echo plus($a, $b);
```

生成的 AST 抽象语法树：
```
AST_STMT_LIST
    0: AST_FUNC_DECL
        flags: 0
        name: "plus"
        docComment: null
        params: AST_PARAM_LIST
            0: AST_PARAM
                flags: 0
                type: null
                name: "a"
                default: null
            1: AST_PARAM
                flags: 0
                type: null
                name: "b"
                default: null
        stmts: AST_STMT_LIST
            0: AST_RETURN
                expr: AST_BINARY_OP
                    flags: BINARY_ADD (1)
                    left: AST_VAR
                        name: "a"
                    right: AST_VAR
                        name: "a"
        returnType: null
        __declId: 0
    1: AST_ASSIGN
        var: AST_VAR
            name: "a"
        expr: 1
    2: AST_ASSIGN
        var: AST_VAR
            name: "b"
        expr: AST_PRE_INC
            var: AST_VAR
                name: "a"
    3: AST_ECHO
        expr: AST_CALL
            expr: AST_NAME
                flags: NAME_NOT_FQ (1)
                name: "plus"
            args: AST_ARG_LIST
                0: AST_VAR
                    name: "a"
                1: AST_VAR
                    name: "b"
```

编译为 `OPCODE` 数组列表：
```
number of ops:  10
compiled vars:  !0 = $a, !1 = $b
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   NOP                                                      
   7     1        ASSIGN                                                   !0, 1
   8     2        PRE_INC                                          $3      !0
         3        ASSIGN                                                   !1, $3
  10     4        INIT_FCALL                                               'plus'
         5        SEND_VAR                                                 !0
         6        SEND_VAR                                                 !1
         7        DO_UCALL                                         $5      
         8        ECHO                                                     $5
         9      > RETURN                                                   1
```

事实上，所有的 PHP 语句都会被编译为 `OPCODE` ，然后交由 Zend 引擎一条条执行，PHP 7.0.12 版本总共定义了 168 类 `OPCODE`：
```
Zend/zend_vm_`opcode`s.h

#define ZEND_NOP                               0
#define ZEND_ADD                               1
#define ZEND_SUB                               2
#define ZEND_MUL                               3
#define ZEND_DIV                               4
#define ZEND_MOD                               5
#define ZEND_SL                                6
#define ZEND_SR                                7
#define ZEND_CONCAT                            8
#define ZEND_BW_OR                             9
#define ZEND_BW_AND                           10
#define ZEND_BW_XOR                           11
#define ZEND_BW_NOT                           12
#define ZEND_BOOL_NOT                         13
#define ZEND_BOOL_XOR                         14
#define ZEND_IS_IDENTICAL                     15
#define ZEND_IS_NOT_IDENTICAL                 16
#define ZEND_IS_EQUAL                         17
#define ZEND_IS_NOT_EQUAL                     18
#define ZEND_IS_SMALLER                       19
#define ZEND_IS_SMALLER_OR_EQUAL              20
#define ZEND_CAST                             21
#define ZEND_QM_ASSIGN                        22
#define ZEND_ASSIGN_ADD                       23
#define ZEND_ASSIGN_SUB                       24
#define ZEND_ASSIGN_MUL                       25
#define ZEND_ASSIGN_DIV                       26
#define ZEND_ASSIGN_MOD                       27
#define ZEND_ASSIGN_SL                        28
#define ZEND_ASSIGN_SR                        29
#define ZEND_ASSIGN_CONCAT                    30
#define ZEND_ASSIGN_BW_OR                     31
#define ZEND_ASSIGN_BW_AND                    32
#define ZEND_ASSIGN_BW_XOR                    33
#define ZEND_PRE_INC                          34
#define ZEND_PRE_DEC                          35
#define ZEND_POST_INC                         36
#define ZEND_POST_DEC                         37
#define ZEND_ASSIGN                           38
#define ZEND_ASSIGN_REF                       39
#define ZEND_ECHO                             40
...
...
...
```

以 `ZEND_ECHO`为例，将交由Zend虚拟机的函数 `ZEND_ECHO_SPEC_TMPVAR_HANDLER` 执行
```

Zend/zend_vm_execute.h #40435

static ZEND_`OPCODE`_HANDLER_RET ZEND_FASTCALL ZEND_ECHO_SPEC_TMPVAR_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
	USE_OPLINE
	zend_free_op free_op1;
	zval *z;

	SAVE_OPLINE();
	z = _get_zval_ptr_var(opline->op1.var, execute_data, &free_op1);

	if (Z_TYPE_P(z) == IS_STRING) {
		zend_string *str = Z_STR_P(z);

		if (ZSTR_LEN(str) != 0) {
			zend_write(ZSTR_VAL(str), ZSTR_LEN(str)); // 将字符输出到控制台 ！！！
		}
	} else {
		zend_string *str = _zval_get_string_func(z);

		if (ZSTR_LEN(str) != 0) {
			zend_write(ZSTR_VAL(str), ZSTR_LEN(str));
		} else if ((IS_TMP_VAR|IS_VAR) == IS_CV && UNEXPECTED(Z_TYPE_P(z) == IS_UNDEF)) {
			GET_OP1_UNDEF_CV(z, BP_VAR_R);
		}
		zend_string_release(str);
	}

	zval_ptr_dtor_nogc(free_op1);
	ZEND_VM_NEXT_`OPCODE`_CHECK_EXCEPTION();
}
```

所以，PHP 引擎本身只是一个 C 语言开发的程序，我们写的 PHP 源代码对引擎来说只是一段输入文本。

PHP 引擎在完成词法/语法解析后，将 PHP 源码编译为一条条的 `OPCODE` ，再交由相应的函数完成执行，获得最终输出。

以上便是一段 PHP 代码的执行之旅，我们还要关注更多细节：

- PHP 变量是无类型的，在引擎内部是如何表示和存储的呢？
- 数组、类、对象、函数、命名空间、继承、接口等概念是如何实现的呢？
- 顺序、条件、循环等语言结构是怎么编译为 `OPCODE` 的呢？
- php-fpm 和 cli 模式有什么异同呢？
- ......

带着这些问题，让我们开启 PHP 内核剖析之旅吧！
