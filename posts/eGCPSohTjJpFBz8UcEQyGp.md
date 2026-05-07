# 使用 Lark 构建自定义文法解析器

最近写了个作业需要用到表达式解析，考虑到自己写不出带错误恢复的自动机，于是遂选择直接导包。*这就是 Python 的生态啊（挺胸）*

```bash
uv add lark
```

用**后缀表达式**的例子来说一下这个包如何使用吧，如果你也要自定义文法然后解析的话。

## 定义文法与语法分析

尽管说后缀表达式用栈就能进行解析，但是还是给一个完整的文法定义吧：

```text
expr -> NUMBER 
      | expr expr OP
```

文法里有**左递归**，管他呢，Lark 的 LALR 解析会处理好的，而且左递归还省栈空间呢。

其中终结符 `NUMBER` 是操作数，`OP` 是加减乘除。
Lark 使用一种魔改过的 **EBNF** 语法声明文法。

> Lark 所用的 EBNF 书写：[Grammar Reference](https://lark-parser.readthedocs.io/en/latest/grammar.html)

有一些特殊的点我会在样例中说明。

```py
from lark import Lark, Token, Tree

GRAMMAR = r"""
?start: expr

expr: NUMBER                -> number
    | expr expr OP          -> operation

NUMBER: /-?[0-9]+(\.[0-9]*)?/
OP: "+" | "-" | "*" | "/"

%import common.WS
%ignore WS
"""


def lexer(stmt: str) -> Tree[Token]:
    parser = Lark(GRAMMAR, start="start", parser="lalr")
    result: Tree[Token] = parser.parse(stmt)
    return result

token_tree = lexer("1 2 + 3 *")
print(token_tree)
```

这里对额外的 Lark 规则进行阐述。

`ignore WS` 代表忽略空格（WhiteSpace），前面 `import` 正是从 Lark 自带的规则库中导入。
每一个产生式后面的箭头 `-> number` 与 `-> operation` 代表这个规则的别名，可以给后面的表达式树构建逻辑中的函数名做区分。
`?start` 前面的问号代表 Lark 会对这个规则（产生式）内联到父级的语法树节点中，不额外生成这个层级的节点。

> Lark 构建树的时候遵循的额外规则：[Tree Construction Reference](https://lark-parser.readthedocs.io/en/latest/tree_construction.html)

最终会输出这种东西（手动加了空格方便读）：

```text
Tree('operation', [
    Tree('operation', [
        Tree('number', [
            Token('NUMBER', '1')
        ]), 
        Tree('number', [
            Token('NUMBER', '2')
        ]), 
        Token('OP', '+')
    ]), 
    Tree('number', [
        Token('NUMBER', '3')
    ]),
    Token('OP', '*')
])
```

## 遍历表达式树构建 AST

在上一个流程中我们获取了一个 `Tree[Token]`，Lark 提供了一个很方便的工具可以让我们沿着表达式树遍历，比如说转换成自己类型的表达式树，或是直接计算结果。

先定义节点类型：

```py
from typing import Literal, TypedDict

class NumNode(TypedDict):
    type: Literal["number"]
    value: float

class OpNode(TypedDict):
    type: Literal["operator"]
    operator: str
    left: ASTNode
    right: ASTNode

ASTNode = NumNode | OpNode
```

只要继承 Lark 的 `Transformer` 类就行了：

```py
# 需要一个额外的处理队列对象来做遍历过程中的类型标注
QueryNode = ASTNode | Token

class BuildAST(Transformer):
    def number(self, items: list[QueryNode]) -> NumNode:
        token = items[0]
        assert isinstance(token, Token) # 检查 token 是否确实是 Token
        return {
            "type": "number",
            "value": float(token.value)
        }

    def operation(self, items: list[QueryNode]) -> OpNode:
        left, right, op = items
        assert isinstance(op, Token)    # 检查 op 是否确实是 Token

        # 检查左右节点是否是已经处理好的类型（包含于 ASTNode 内） 
        assert not isinstance(left, Token)  
        assert not isinstance(right, Token)
        return {
            "type": "op",
            "operator": str(op),
            "left": left,
            "right": right
        }

transformer = BuildAST()
print(transformer.transform(token_tree))
```

断言是为了方便调试，如果没有 Lark 只会在语法错误的时候给一个未知位置的 `AttributeError`，所以得靠断言来确定具体的错误位置。

这时候就是输出可读可二次处理的对象了（手动加了空格方便读）：

```text
{
    'type': 'op', 
    'operator': '*', 
    'left': {
        'type': 'op',
        'operator': '+',
        'left': {
            'type': 'number',
            'value': 1.0
        },
        'right': {
            'type': 'number',
            'value': 2.0
        }
    }, 
    'right': {
        'type': 'number',
        'value': 3.0
    }
}
```

## 类型限定更强的 Transformer

```py
from typing import TypedDict, Union, cast
from lark import Transformer, Token

class NumNode(TypedDict):
    type: Literal["number"]
    value: float

class OpNode(TypedDict):
    type: Literal["operator"]
    operator: str
    left: "ASTNode"  # 使用引号处理递归引用
    right: "ASTNode"

ASTNode = Union[NumNode, OpNode]

class BuildAST(Transformer):
    def number(self, items: list[Token]) -> NumNode:
        # 限定 items 内部必须是 Token
        token = items[0]
        assert isinstance(token, Token)
        return {
            "type": "number",
            "value": float(token.value)
        }

    def operation(self, items: list[Union[ASTNode, Token]]) -> OpNode:
        match items:
            # 根据文法: operation -> expr expr OP  
            # items[0] 和 items[1] 必定是已经计算好的 ASTNode
            # items[2] 必定是 Token
            case [left, right, Token() as op]:
                return {
                    "type": "operator",
                    "operator": str(op.value),
                    # 用 cast 让类型检查强制认定类型是 ASTNode
                    "left": cast(ASTNode, left),
                    "right": cast(ASTNode, right)
                }
            case _:
                # 这里可以给一个更具体的错误，都已经精确到一个具体的产生式了
                raise ValueError(f"Synax Error as: {items}")
```
