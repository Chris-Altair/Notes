### 1.原理

设计一个DSL（领域特定语言）时，词法分析和语法分析是两个重要步骤。以下是这两个步骤的基本流程：

1. 词法分析（Lexical Analysis）
   词法分析的主要任务是将源代码字符串分解成一个个“记号”（token）。具体步骤包括：

定义词法规则：确定语言中的基本构件，比如关键字、标识符、操作符、常量等。可以使用正则表达式来描述这些规则。

创建词法分析器：可以手动编写，或者使用工具（如 Lex 或 Flex）生成词法分析器。词法分析器会扫描源代码，根据定义的词法规则生成记号流。

处理错误：设计机制来处理词法错误，比如未知的字符或不匹配的记号。

2. 语法分析（Syntax Analysis）
   语法分析的任务是根据语言的语法规则，将记号流转换为一个抽象语法树（AST）。具体步骤包括：

定义语法规则：使用上下文无关文法（CFG）来描述语言的语法结构。可以使用巴科斯-诺尔范式（BNF）来表示。

创建语法分析器：可以选择自顶向下（如递归下降解析）或自底向上（如 LR 解析）的方法。工具如 Yacc 或 Bison 可以帮助生成解析器。

构建抽象语法树：在解析过程中，根据语法规则构建 AST，AST 节点表示程序的结构和操作。

错误处理：实现语法错误检测和恢复机制，以便在发现错误时给出适当的反馈。

总结
设计 DSL 时，首先要明确语言的语义和目标，然后根据这些设计词法和语法规则，最后实现词法分析器和语法分析器。你可以使用现有的工具和库来简化这个过程。

### 2.示例

有个需求需要将数据按一定规则转成银行需要的特定格式的文本文件。

所以需要设计一个模板规则，简单的规则可以使用正则去做，复杂的还是通过编译原理相关的知识解决较好。

模板是这种格式

```php
<listName>$[fieldName1]|$[fieldName2]</listName>
```

#### ①词法分析

首先明确模板中的token都有什么，显然token可以分为如下类别

```java
enum TemplateTokenType {
    /* < */
    LEFT_START_TEMPLATE_SYMBOL,
    /* </ */
    LEFT_END_TEMPLATE_SYMBOL,
    /* > */
    RIGHT_TEMPLATE_SYMBOL,
    /* $[ */
    LEFT_FIELD_SYMBOL,
    /* ] */
    RIGHT_FIELD_SYMBOL,
    /* | */
    SEPARATOR,
    /* 字段名：字母开头后面接字母、数字、下划线 */
    NAME,
}
```



```java
interface TemplateConstants {
    String LEFT_START_TEMPLATE_SYMBOL = "<";
    String LEFT_END_TEMPLATE_SYMBOL = "</";
    String RIGHT_TEMPLATE_SYMBOL = ">";
    
    String LEFT_FIELD_SYMBOL = "$[";
    String RIGHT_FIELD_SYMBOL = "]";
    
    String SEPARATOR = "|";
    
}
```

词法分析会将字符串解析为token的序列，如果字符串不满足构词规则会直接报错（公司电脑有加密代码就简写了，重写一份太麻烦。。。）

方法核心就是不断读取字符看是否匹配对应词法，匹配则加入列表，否则报错

```java
class TemplateLexer {
    private CharReader charReader;
    class CharReader {
        ...
    }
    
    public List<TemplateToken> tokenize() {
        List<TemplateToken> tokens = new ArrayList<>();
        while(charReader.next()) {
            char c = charReader.next();
            switch(c) {
                case '<':{
                    //判断是<还是</，并添加token列表
                }
                case '>':{
                    //添加token列表
                }
                case '$':{
                    //判断是否满足$[，满足则添加token列表
                }
                case ']':{
                    //添加token列表
                }
                case '|':{
                    //添加token列表
                }
                case ' ':{
                    //空格跳过
                }
                default:{
                    //解析并判断是否符合字段名，满足则添加token列表
                }
            }
        }
        return tokens;
    }
}
```



#### ②语法分析

词法分析后已经拿到token序列了，接下来就需要判断序列是否满足我们设计的语法了，首先就是需要明确我们的语法规则（其实这是写代码之前就应该设计好的），可以用BNF表达式表示

BNF表达式如下：

```php
<template> ::= <start> <fieldList> <end>
<fieldList> ::= <field> <pair>*
<pair> ::= <separator> <field>

<separator> ::= "|"

<start> ::= "<" <name> ">"

<end> ::= "</" <name> ">"

<field> ::= "$[" <name> "]"

<name> ::= <letter> <letterOrDigit>*
<letter> ::= a | b | c | ... | z | A | B | C | ... | Z | _
<digit> ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
<letterOrDigit> ::= <letter> | <digit>
```

核心逻辑就是比较当前token是否满足期望，是的化就指针后移，否则就报错

TemplateVO就是解析后的我们需要的实体

```java
class TemplateParser {
    private TokenReader tokenReader;
    class TokenReader {
        ...
    }
    public TemplateParser(List<TemplateToken> tokens) {
        //构造TokenReader
    }
    
    public TemplateVO paseTokens() {
        String listStartName = parseStart();
        List<String> fieldList = parseFieldList();
        String listEndName = parseEnd();
        ...
    }
    
    private void match(TemplateTokenType expectedTokenType) {
        //校验当前token是否符合期望的token，并将指针+1
    }
    
    private List<String> parseFieldList() {
        parseField();
        while(...) {
            parseSeparator();
            parseField();
        }
    }
    
    private String parseField() {
        match(TemplateTokenType.LEFT_FIELD_SYMBOL);
        parseName();
        match(TemplateTokenType.RIGHT_FIELD_SYMBOL);
    }
    
    private String parseStart() {
        match(TemplateTokenType.LEFT_START_TEMPLATE_SYMBOL);
        parseName();
        match(TemplateTokenType.RIGHT_TEMPLATE_SYMBOL);
    }
    
    ...
}
```

