之前在大神Douglas Crockford的博客上，翻出来一篇很久远的关于如何实现一个js parser的[文章](http://crockford.com/javascript/tdop/tdop.html)。结合网上其他大牛的解读，修改成如下的版本，留作参考

该算法基于Vaughan Pratt的Top Down Operator Precedence算法

```typescript
// ast node type
enum NodeType {
    NAME,
    LITERAL,
    UNARY,
    BINARY,
    TERNARY,
    PROPERTY,
    ASSIGNMENT
}

// ast node interface
interface INode {
    type: NodeType,
    value: string,
    children: Array<INode | null>
}

// lex token type
enum TokenType {
    END,
    NAME,
    LITERAL,
    SUM,        // +
    SUB,        // -
    MUL,        // *
    DIV,        // /
    INC,        // ++
    DEC,        // --
    AND,        // &&
    OR,         // ||
    NOT,        // !
    BITAND,     // &
    BITOR,      // |
    BITXOR,     // ^
    BITNOT,     // ~
    EQ,         // ==
    SEQ,        // ===
    NEQ,        // !=
    SNEQ,       // !==
    GT,         // >
    GTE,        // >=
    LT,         // <
    LTE,        // <=
    QUESTION,   // ?
    COLON,      // :
    DOT,        // .
    ASSIGN,     // =
    SUMASSIGN,  // +=
    SUBASSIGN,  // -=
    MULASSIGN,  // *=
    DIVASSIGN,  // /=
    LEFTPAREN,  // (
    RIGHTPAREN, // )
    LEFTSQUARE, // [
    RIGHTSQUARE // ]
}

// lex token interface
interface IToken {
    type: TokenType,
    value: string
}

// operator precedence
enum BindPower {
    NOBINDING = 0,
    ASSIGNMENT = 10,
    CONDITIONAL = 20,
    LOGICAL = 30,
    RELATIONAL = 40,
    SUMSUB = 50,
    MULDIV = 60,
    PREFIX = 70,
    SUFFIX = 80,
    OTHER = 90
}

// parser interface
interface IParser {
    parse(rightBindPower: number): INode,
    seek(type: TokenType): IToken
}
    
interface ISymbol {
    leftBindPower: number
}

// null denotation
interface INudSymbol extends ISymbol {
    parse(parser: IParser, token: IToken): INode
}

// left denotation
interface ILedSymbol extends ISymbol {
    parse(parser: IParser, token: IToken, left: INode): INode
} 

class ASTNode implements INode {
    type: NodeType
    value: string
    children: Array<INode | null>

    constructor(type: NodeType, value: string, ...children: Array<INode | null>) {
        this.type = type
        this.value = value
        this.children = children
    }
}

class NameSymbol implements INudSymbol {
    leftBindPower: number = BindPower.NOBINDING

    parse(parser: IParser, token: IToken): INode {
        return new ASTNode(NodeType.NAME, token.value)
    }
}

class LiteralSymbol implements INudSymbol {
    leftBindPower: number = BindPower.NOBINDING

    parse(parser: IParser, token: IToken): INode {
        return new ASTNode(NodeType.LITERAL, token.value)
    }
}

class AssignmentSymbol implements ILedSymbol {
    leftBindPower: number = BindPower.ASSIGNMENT

    parse(parser: IParser, token: IToken, left: INode): INode {
        if (left.type !== NodeType.PROPERTY, left.type !== NodeType.NAME) {
            throw Error("Bad lvalue")
        }

        return new ASTNode(NodeType.ASSIGNMENT, token.value, left, parser.parse(this.leftBindPower - 1))
    }
}

class ConditionalSymbol implements ILedSymbol {
    leftBindPower: number = BindPower.CONDITIONAL

    parse(parser: IParser, token: IToken, left: INode): INode {
        const thenArm: INode = parser.parse(BindPower.NOBINDING)
        parser.seek(TokenType.COLON)
        const elseArm: INode = parser.parse(this.leftBindPower - 1)

        return new ASTNode(NodeType.TERNARY, token.value, left, thenArm, elseArm)
    }
}

class PropertySymbol implements ILedSymbol {
    leftBindPower: number = BindPower.OTHER

    parse(parser: IParser, token: IToken, left: INode): INode {
        const propName: IToken = parser.seek(TokenType.NAME)
        const propNode = new ASTNode(NodeType.NAME, propName.value)

        return new ASTNode(NodeType.PROPERTY, token.value, left, propNode)
    }
}

class AccessSymbol implements ILedSymbol {
    leftBindPower: number = BindPower.OTHER

    parse(parser: IParser, token: IToken, left: INode): INode {
        const subExpr = parser.parse(BindPower.NOBINDING)
        parser.seek(TokenType.RIGHTSQUARE)

        return new ASTNode(NodeType.PROPERTY, token.value, left, subExpr)
    }
}

class GroupSymbol implements INudSymbol {
    leftBindPower: number = BindPower.OTHER

    parse(parser: IParser, token: IToken): INode {
        const subExpr = parser.parse(BindPower.NOBINDING)
        parser.seek(TokenType.RIGHTPAREN)

        return subExpr
    }
}

class PrefixOperatorSymbol implements INudSymbol {
    leftBindPower: number = BindPower.PREFIX

    parse(parser: IParser, token: IToken): INode {
        return new ASTNode(NodeType.UNARY, token.value, null, parser.parse(this.leftBindPower))
    }
}

class SuffixOperatorSymbol implements ILedSymbol {
    leftBindPower: number = BindPower.SUFFIX

    parse(parser: IParser, token: IToken, left: INode): INode {
        return new ASTNode(NodeType.UNARY, token.value, left, null)
    }
}

class BinaryOperatorSymbol implements ILedSymbol {
    // dynmic bind power
    leftBindPower: number
    isRight: boolean

    constructor(leftBindPower: number, isRight: boolean = false) {
        this.leftBindPower = leftBindPower
        this.isRight = isRight
    }

    parse(parser: IParser, token: IToken, left: INode): INode {
        return new ASTNode(NodeType.BINARY, token.value, left, parser.parse(this.leftBindPower - (this.isRight ? 1 : 0)))
    }
}

// pratt parser core    
class Parser implements IParser {
    // nud lookup table
    private nudSymbolTable: { [key: number]: INudSymbol } = {}
    // led lookup table
    private ledSymbolTable: { [key: number]: ILedSymbol } = {}

    // token stream
    private tokens: Array<IToken>
    // look ahead token LL(1)
    private ahead: IToken | null

    constructor(tokens: Array<IToken>) {
        this.tokens = tokens
        this.ahead = null
    }

    private leftBindPower(): number {
        const token = this.peak()
        const symbol = this.ledSymbolTable[token.type]

        return symbol ? symbol.leftBindPower : 0
    }

    parse(rightBindPower: number = BindPower.NOBINDING): INode {
        let left

        const token = this.next()
        const symbol = this.nudSymbolTable[token.type]

        left = symbol.parse(this, token)

        while (rightBindPower < this.leftBindPower()) {
            const token = this.next()
            const symbol = this.ledSymbolTable[token.type]

            left = symbol.parse(this, token, left)
        }

        return left
    }

    // register nud symbol
    prefix(type: TokenType, symbol: INudSymbol): void {
        this.nudSymbolTable[type] = symbol
    }

    // register led symbol
    infix(type: TokenType, symbol: ILedSymbol): void {
        this.ledSymbolTable[type] = symbol
    }
    
    seek(type: TokenType): IToken {
        const token = this.peak()

        if (token.type !== type) {
            throw new Error(`Expected ${ TokenType[type] }`)
        }

        return this.next()
    }

    next(): IToken {
        let token: IToken | undefined

        if (this.ahead) {
            token = this.ahead
            this.ahead = null
        } else {
            token = this.tokens.shift()
        }

        return token ? token : { type: TokenType.END, value: "" }
    }

    peak(): IToken {
        return this.ahead = this.next()
    }
}

// expression syntax definition
class ExpressionParser extends Parser {
    constructor(tokens: Array<IToken>) {
        super(tokens)

        // hello
        this.prefix(TokenType.NAME, new NameSymbol())
        // 'hello'
        this.prefix(TokenType.LITERAL, new LiteralSymbol())

        // a = b
        this.infix(TokenType.ASSIGN, new AssignmentSymbol())
        // a += b
        this.infix(TokenType.SUMASSIGN, new AssignmentSymbol())
        // a -= b
        this.infix(TokenType.SUBASSIGN, new AssignmentSymbol())
        // a *= b
        this.infix(TokenType.MULASSIGN, new AssignmentSymbol())
        // a /= b
        this.infix(TokenType.DIVASSIGN, new AssignmentSymbol())

        // a ? b : c
        this.infix(TokenType.QUESTION, new ConditionalSymbol())

        // a && b
        this.infix(TokenType.AND, new BinaryOperatorSymbol(BindPower.LOGICAL, true))
        // a || b
        this.infix(TokenType.OR, new BinaryOperatorSymbol(BindPower.LOGICAL, true))
        // a & b
        this.infix(TokenType.BITAND, new BinaryOperatorSymbol(BindPower.LOGICAL, true))
        // a | b
        this.infix(TokenType.BITOR, new BinaryOperatorSymbol(BindPower.LOGICAL, true))
        // a ^ b
        this.infix(TokenType.BITXOR, new BinaryOperatorSymbol(BindPower.LOGICAL, true))

        // a == b
        this.infix(TokenType.EQ, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a === b
        this.infix(TokenType.SEQ, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a != b
        this.infix(TokenType.NEQ, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a !== b
        this.infix(TokenType.SNEQ, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a > b
        this.infix(TokenType.GT, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a >= b
        this.infix(TokenType.GTE, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a < b
        this.infix(TokenType.LT, new BinaryOperatorSymbol(BindPower.RELATIONAL))
        // a <= b
        this.infix(TokenType.LTE, new BinaryOperatorSymbol(BindPower.RELATIONAL))

        // a + b
        this.infix(TokenType.SUM, new BinaryOperatorSymbol(BindPower.SUMSUB))
        // a - b
        this.infix(TokenType.SUB, new BinaryOperatorSymbol(BindPower.SUMSUB))

        // a * b
        this.infix(TokenType.MUL, new BinaryOperatorSymbol(BindPower.MULDIV))
        // a / b
        this.infix(TokenType.DIV, new BinaryOperatorSymbol(BindPower.MULDIV))

        // +a
        this.prefix(TokenType.SUM, new PrefixOperatorSymbol())
        // -a
        this.prefix(TokenType.SUB, new PrefixOperatorSymbol())
        // !a
        this.prefix(TokenType.NOT, new PrefixOperatorSymbol())
        // ~a
        this.prefix(TokenType.BITNOT, new PrefixOperatorSymbol())

        // ++a
        this.prefix(TokenType.INC, new PrefixOperatorSymbol())
        // --a
        this.prefix(TokenType.DEC, new PrefixOperatorSymbol())

        // a++
        this.infix(TokenType.INC, new SuffixOperatorSymbol())
        // a--
        this.infix(TokenType.DEC, new SuffixOperatorSymbol())

        // a.b
        this.infix(TokenType.DOT, new PropertySymbol())
        // a[b]
        this.infix(TokenType.LEFTSQUARE, new AccessSymbol())

        // (a + b)
        this.prefix(TokenType.LEFTPAREN, new GroupSymbol())
    }
}
```