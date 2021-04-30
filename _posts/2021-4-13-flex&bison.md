---
title: "Compilers: A Parser with Flex & Bison"
categories:
  - PL
---

Report for a small compiler project, introducing basic flex and bison.

## flex & bison的工作原理

bison需要和flex协作对文本完成词法和语法的分析，如下图：

![workflow]({{ site.url }}{{ site.baseurl }}/assets/images/workflow.png){: .align-center}

其中，\*.y 是为bison/yacc描述语法分析的文件，\*.l 是为flex描述词法分析的文件。这其中，首先需要定义语法和它们对应的规则，例如下方代码描述了方法声明(MethodDecl)的四种规则，分别对应是否为主函数，是否存在参数的可能：

```c++
MethodDecl:
        Type MAIN Id '(' FormalParams ')' Block 
            { $$=new MethodDecl(7, $1, $2, $3, $4, $5, $6, $7); }
    |   Type      Id '(' FormalParams ')' Block 
            { $$=new MethodDecl(6, $1, $2, $3, $4, $5, $6); }
    |   Type MAIN Id '('              ')' Block 
            { $$=new MethodDecl(6, $1, $2, $3, $4, $5, $6); }
    |   Type      Id '('              ')' Block 
            { $$=new MethodDecl(5, $1, $2, $3, $4, $5); }
        ;
```

有了语法定义之后，才进一步设计 \*.l 来定义词法。这是因为，定义了语法规则之后，解析词法所需要的token类型就确定了，例如上方的MAIN表示主函数，Type表示一个类型声明。因此flex文件需要通过包含bison产生的头文件 \*.tab.hh 来确定需要解析的token类型，flex和bison通过这种联系来进行协作：

```c++
/* Token type.  */
#ifndef YYTOKENTYPE
# define YYTOKENTYPE
  enum yytokentype
  {
    INum = 258,
    INT = 259,
    REAL = 260,
    STRING = 261,
    FNum = 262,
    QString = 263,
    WRITE = 264,
    READ = 265,
    IF = 266,
    ELSE = 267,
    RETURN = 268,
    BEGINN = 269,
    END = 270,
    MAIN = 271,
    Id = 272,
    ASSIGN = 273,
    EQ = 274,
    NEQ = 275
  };
#endif
```

每个token都是一个int(因此lexer定义为int yylex())。yylex()扫描输入文本，每次调用返回文字对应的token类型，bison生成的语法分析yyparse()则使用token转移状态，匹配相应的规则，并执行相应的动作。0-255是为单一字符保留的。例如'('，可以直接返回'('本身作为它的token。

最后通过编译器来生成一个可执行程序，yylex()函数实现在 \*.yy.cpp中，调用它的yyparse()函数则实现在 \*.tab.cc中。用户可以自己定义其他的功能，最后一起编译即可。本次实验的Makefile如下：

```makefile
vpath %.h% include
vpath tiny_% src

YACC=bison
LEX=flex

CXX=g++

tiny_parser: tiny_lexer.yy.cpp tiny_parser.tab.cc tiny_parser.tab.hh
    $(CXX) -o tiny_parser -g -O3 src/tiny_lexer.yy.cpp src/tiny_parser.tab.cc src/tiny.cpp

tiny_parser.tab.cc tiny_parser.tab.hh: tiny_parser.y
    $(YACC) -t -v -d -o src/tiny_parser.tab.cc src/tiny_parser.y
    mv src/*.h* src/*.output include

tiny_lexer.yy.cpp: tiny_lexer.l tiny_parser.tab.hh
    $(LEX) -o src/tiny_lexer.yy.cpp src/tiny_lexer.l

clean:
    -rm tiny_parser include/*.output
```

## flex的使用

### 基本构成

flex的详细介绍： [https://www.cs.princeton.edu/~appel/modern/c/software/flex/flex.html](https://www.cs.princeton.edu/~appel/modern/c/software/flex/flex.html)

flex的主要内容是编写正则表达式，从而对文本进行匹配。主要的内容都是如下形式的：

```c++
 Patterns     Actions
"WRITE"     { yycout(); yylval.nptr=new Terminal("WRITE"); return WRITE; }
```

左侧是一个正则表达式，右侧是匹配到这个正则表达式所产生的动作。一般来说，这个动作至少需要返回这个表达式对应的token类型，也就是上方bison定义的其中一个类型。此外，还可以执行其他动作：上面的例子中还包括输出提示信息(yycout())，以及给语义值yylval赋值。

flex中有几个关键的变量，它们是记录着yylex()的部分状态：

* yytext：一个字符指针，指向当前匹配到的字符串。
* yyleng：yytext指向的字符串的长度。
* yylval：和yytext所关联的语义值，可以给bison所使用。

### yylval的作用

这里比较重要的是yylval。在bison进行语法分析的时候，yyparse()会维护一个栈，这个栈中的每一个元素的值，都是由yylval所指定的。不过，这里产生了一个问题，不同token类型关联的语义值类型也是不同的，这怎么办呢？bison要求yylval是一个用户定义的union类型，通过不同的分量来存储和访问不同类型的语义值：

```c++
%union {
    int ival;
    float fval;
    string* sval;
    node* nptr;
}
```

于是另外一个问题出现了，union的每个分量最好是存放空间固定的类型——否则以C为输出语言的代码文件大概率是有问题的；即使改为输出C++，在分量之间切换也会带来额外的构造/析构。因此上面定义的复杂类型都是用指针表示，通过new一个对象来赋值。

yylval之所以方便，是因为我们可以在lex中就定义好特定文本对应的语义值，然后在语法规则中分别处理它们。之前尝试过在语法分析时才生成语义值，由于一个token会出现在多个语法规则中，生成语义值的代码就会重复，导致臃肿。

### 注释和字符串的匹配

有些匹配规则非常难表达。例如多行注释，上方的flex介绍中有一个借助input()函数（弹出yytext之后的第一个字符）实现的过滤方式。通过查找资料，找到了一个借助condition的实现方式：

```c++
%x C_COMMENT

%%

"/**"            { yycout(); BEGIN(C_COMMENT); }
<C_COMMENT>"**/" { yycout(); BEGIN(INITIAL); }
<C_COMMENT>.     { }
<C_COMMENT>\n    { }
```

字符串的匹配也很难想到，同样是上网查询了解的：

```c++
string                      \"([^\\\"]|\\.)*\"
```

## bison的使用

bison的主体构成也是类似的规则-动作结构：

```c++
// Specify which type of yylval should be used
%token <nptr> WRITE READ IF ELSE RETURN BEGINN END MAIN Id
%right <nptr> ASSIGN
%left  <nptr> EQ NEQ 
%left  <nptr> '+' '-' '*' '/'
%token <nptr> ';' ',' '(' ')'

// an example of rule-action pairs
MethodDecl:
        Type MAIN Id '(' FormalParams ')' Block 
            { $$=new MethodDecl(7, $1, $2, $3, $4, $5, $6, $7); }
    |   Type      Id '(' FormalParams ')' Block 
            { $$=new MethodDecl(6, $1, $2, $3, $4, $5, $6); }
    |   Type MAIN Id '('              ')' Block 
            { $$=new MethodDecl(6, $1, $2, $3, $4, $5, $6); }
    |   Type      Id '('              ')' Block 
            { $$=new MethodDecl(5, $1, $2, $3, $4, $5); }
        ;
```

上面的左侧，是MethodDecl的四种匹配规则，右侧是每个匹配对应的动作。以'\|'进行分隔。在上一部分，我们提到了yylval结构是语义值栈的数据类型，bison定义了一些符号，来方便用户操作语义值栈中的数据：

* \$\$: 归约之后被重新压入栈中的元素
* \$i: 归约之前栈中距离栈顶编号为i的元素

也就是说，\$\$对应被归约的MethodDecl，\$3对应Id或者左括号'('（如果没有MAIN）。bison产生cpp代码文件的时候，只是对这些符号进行了文本替换。至于这些符号替换成yylval的哪一个分量，则由文件上方的%token, %type, %left旁边的<*type*>进行指定，生成的代码是这样的：

```c++
 case 7:
#line 82 "src/tiny_parser.y"
    { (yyval.nptr)=new MethodDecl(7, (yyvsp[-6].nptr), (yyvsp[-5].nptr), (yyvsp[-4].nptr), (yyvsp[-3].nptr), (yyvsp[-2].nptr), (yyvsp[-1].nptr), (yyvsp[0].nptr)); }
#line 1443 "src/tiny_parser.tab.cc"
    break;

  case 8:
#line 83 "src/tiny_parser.y"
    { (yyval.nptr)=new MethodDecl(6, (yyvsp[-5].nptr), (yyvsp[-4].nptr), (yyvsp[-3].nptr), (yyvsp[-2].nptr), (yyvsp[-1].nptr), (yyvsp[0].nptr)); }
#line 1449 "src/tiny_parser.tab.cc"
    break;

  case 9:
#line 84 "src/tiny_parser.y"
    { (yyval.nptr)=new MethodDecl(6, (yyvsp[-5].nptr), (yyvsp[-4].nptr), (yyvsp[-3].nptr), (yyvsp[-2].nptr), (yyvsp[-1].nptr), (yyvsp[0].nptr)); }
#line 1455 "src/tiny_parser.tab.cc"
    break;

  case 10:
#line 85 "src/tiny_parser.y"
    { (yyval.nptr)=new MethodDecl(5, (yyvsp[-4].nptr), (yyvsp[-3].nptr), (yyvsp[-2].nptr), (yyvsp[-1].nptr), (yyvsp[0].nptr)); }
#line 1461 "src/tiny_parser.tab.cc"
    break;
```

## TINY语言的语法树

TINY语言的定义：[http://jlu.myweb.cs.uwindsor.ca/214/language.htm](http://jlu.myweb.cs.uwindsor.ca/214/language.htm)

按照这个定义就可以写出大部分bison规则了。需要注意的是存在闭包的规则，需要转换成递归定义的规则。左右递归都是可行的：

```c++
Block -> BEGIN Statement+ END

// define statements as:
Statements:
        Statement             { $$=new Statements(1, $1); }
    |   Statements Statement  { $$=new Statements(2, $1, $2); }
        ;
```

### 树节点的定义

然后实现语法树。语法树的实现有很多不同的方法，我的实现方法利用了C++的多态特性。在我的设计中，最初的目的是为了方便之后编写对语法树的有关操作。在TINY语言中，语法树中的所有节点都是node类的子类，node中定义了一个node*的向量和一个整数，分别表示node的孩子节点和孩子节点的数量：

```c++
/*
    base class for all nonterminals. 
    This is to provide a unified API to parse the syntax tree. See method print_tree_dfs()
*/
class node {
public:
    node() { Nops=0; }
    virtual ~node() { for(int i = 0;i < Nops;i++) delete ops[i]; }
    virtual string get_node_type() { return "Base"; }

    int Nops;
    vector<node*> ops;
    // # of operands and vector of operands
};

class Program: public node {
public:
    virtual string get_node_type() { return "Program"; }
    Program(int _nops, ...) { Nops=_nops; ops_pushback(); }
};
```

由于节点的存储方式统一，所有节点可以共享析构函数~node()，只需要调用delete root就可以释放整个语法树。所有子类通过虚函数nptr->get_node_type()来返回自己的类型。除了个别节点（终端符号等），所有非终端符号的构造函数都相似，利用stdarg.h提供的可变参数来压入任意数量的孩子节点。于此同时，bison文件中的动作也可以用统一的格式编写，如本段的开头所示。

### 输出语法树

定义好了语法树后，就可以自顶向下输出它的结构。bison文件中，在最后归约的动作中调用输出方法。递归输出语法树的函数在实验一中就已经实现了，正是因为我实现的语法树结构统一，所以基本可以直接搬过来用。

```c++
Program:  
    MethodDecls           { 
        $$=new Program(1, $1); 
    #ifdef SIMPLE_TREE
        eliminate_left_recursion($$);
    #endif
        vector<int> lineStk; print_tree_dfs($$, 0, lineStk); 
        delete $$;
    }
    ;
```

### 消除左递归

编译运行之后，就初步得到了所需的语法树结构。解析样例代码，得到了正确的结果。但是存在一个问题，就是左递归定义的语法规则，导致几种可变长度的语法结构也被迫用递归来表示，如下所示。这会导致复杂代码的语法树深度快速增加。

```text
   |  |- Block
   |  |  |- BEGIN
   |  |  |- Statements
   |  |  |  |- Statements
   |  |  |  |  |- Statements
   |  |  |  |  |  |- Statements
   |  |  |  |  |  |  |- Statements
   |  |  |  |  |  |  |  |- Statements
   |  |  |  |  |  |  |  |  |- Statements
   |  |  |  |  |  |  |  |  |  |- LOCALVARDECL
   |  |  |  |  |  |  |  |  |     |- LocalVarDecl
   |  |  |  |  |  |  |  |  |        |- INT
   |  |  |  |  |  |  |  |  |        |- x
   |  |  |  |  |  |  |  |  |        |- ;
```

不过这个情况容易进行优化。在bison文件中，我的语法规则都是用左递归进行实现。这使得这种情况容易发现：判断每个节点的第一个孩子，其类型是否和根节点相同。如果相同，那么将这个孩子节点的孩子们展开到根节点上。注意要自底向上展开，先进入孩子节点处理，然后处理根节点：

```c++
void eliminate_left_recursion(node* root)
{
    if(root->Nops == 0) return ;

    for(int i = 0;i < root->Nops;i++)
        eliminate_left_recursion(root->ops[i]);

    // a left recursion is found, unfold the left child
    node* lchild = root->ops[0];
    if(lchild->get_node_type() == root->get_node_type()) {
        vector<node*> newOps;
        for(int i = 0;i < lchild->Nops;i++)
            newOps.push_back(lchild->ops[i]);
        for(int i = 1;i < root->Nops;i++)
            newOps.push_back(root->ops[i]);

        root->ops = newOps;
        root->Nops = root->ops.size();
    }
}
```

消除之后的输出结果为：

```text
   |  |- Block
   |  |  |- BEGIN
   |  |  |- Statements
   |  |  |  |- LOCALVARDECL
   |  |  |  |  |- LocalVarDecl
   |  |  |  |     |- INT
   |  |  |  |     |- x
   |  |  |  |     |- ;
```

到这一步，基本就完成了实验的要求。
