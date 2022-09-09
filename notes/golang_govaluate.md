- [govaluate](#govaluate)
  - [parse阶段](#parse阶段)
    - [parseTokens](#parsetokens)
    - [token类型](#token类型)
    - [planStages](#planstages)
    - [operator](#operator)
  - [执行阶段](#执行阶段)
  - [总结](#总结)

# [govaluate](https://github.com/Knetic/govaluate)
govaluate提供了简单的C类似的表达式的求值功能.
```go
    expression, err := govaluate.NewEvaluableExpression("10 > 0");
    result, err := expression.Evaluate(nil);
    // result is now set to "true", the bool value.
```
传参需要用个`map[string]interface{}`来传递.
```go
    expression, err := govaluate.NewEvaluableExpression("(requests_made * requests_succeeded / 100) >= 90");

    parameters := make(map[string]interface{}, 8)
    parameters["requests_made"] = 100;
    parameters["requests_succeeded"] = 80;

    result, err := expression.Evaluate(parameters);
    // result is now set to "false", the bool value.
```
除了返回bool, 也可以返回数字
```go
    expression, err := govaluate.NewEvaluableExpression("(mem_used / total_mem) * 100");

    parameters := make(map[string]interface{}, 8)
    parameters["total_mem"] = 1024;
    parameters["mem_used"] = 512;

    result, err := expression.Evaluate(parameters);
    // result is now set to "50.0", the float64 value.
```
也可以预定义可以执行的函数, 用`govaluate.NewEvaluableExpressionWithFunctions`注册
```go
    functions := map[string]govaluate.ExpressionFunction {
        "strlen": func(args ...interface{}) (interface{}, error) {
            length := len(args[0].(string))
            return (float64)(length), nil
        },
    }

    expString := "strlen('someReallyLongInputString') <= 16"
    expression, _ := govaluate.NewEvaluableExpressionWithFunctions(expString, functions)

    result, _ := expression.Evaluate(nil)
    // result is now "false", the boolean value
```

## parse阶段

```go
func NewEvaluableExpression(expression string) (*EvaluableExpression, error) {

    functions := make(map[string]ExpressionFunction)
    return NewEvaluableExpressionWithFunctions(expression, functions)
        //也要经过解析token
        //在循环里逐个查看字符, 获得token的切片: []ExpressionToken -- 这里的token都不是一个ast
        ret.tokens, err = parseTokens(expression, functions)
        err = checkBalance(ret.tokens)
        err = checkExpressionSyntax(ret.tokens)
        //优化token slice. 
        ret.tokens, err = optimizeTokens(ret.tokens)
        //planStages把token列表转为执行树
        ret.evaluationStages, err = planStages(ret.tokens)
}
```

### parseTokens
对input的每个字符都parse
```go
func newLexerStream(source string) *lexerStream {

    var ret *lexerStream
    var runes []rune

    for _, character := range source {
        runes = append(runes, character)
    }

    ret = new(lexerStream)
    ret.source = runes
    ret.length = len(runes)
    return ret
}
```

### token类型
token是{Kind, Value}
```go
type ExpressionToken struct {
    //Kind是预定义的int标号
    Kind  TokenKind
    //值就是万能interface
    Value interface{}
}

type TokenKind int

const (
    UNKNOWN TokenKind = iota

    PREFIX
    NUMERIC
    BOOLEAN
    STRING
    PATTERN
    TIME
    VARIABLE
    FUNCTION
    SEPARATOR
    ACCESSOR

    COMPARATOR
    LOGICALOP
    MODIFIER

    CLAUSE
    CLAUSE_CLOSE

    TERNARY
)
```

### planStages
从token到执行树. 那么执行树里面有什么呢?
```go
/*
    Creates a `evaluationStageList` object which represents an execution plan (or tree)
    which is used to completely evaluate a set of tokens at evaluation-time.
    The three stages of evaluation can be thought of as parsing strings to tokens, then tokens to a stage list, then evaluation with parameters.
*/
func planStages(tokens []ExpressionToken) (*evaluationStage, error) {

    stream := newTokenStream(tokens)

    stage, err := planTokens(stream)
    if err != nil {
        return nil, err
    }

    // while we're now fully-planned, we now need to re-order same-precedence operators.
    // this could probably be avoided with a different planning method
    reorderStages(stage)

    stage = elideLiterals(stage)
    return stage, nil
}
```

### operator
在plan执行树的时候, 所有最底层的操作都被定义为evaluationStage.operator
```go
/*
    A truly special precedence function, this handles all the "lowest-case" errata of the process, including literals, parmeters,
    clauses, and prefixes.
*/
func planValue(stream *tokenStream) (*evaluationStage, error) {
    ...
    //根据token的kind信息决定operator
    switch token.Kind {
    ...
    case VARIABLE:
        operator = makeParameterStage(token.Value.(string))

    case NUMERIC:
        fallthrough
    case STRING:
        fallthrough
    case PATTERN:
        fallthrough
    case BOOLEAN:
        symbol = LITERAL
        operator = makeLiteralStage(token.Value)
    }
}
```
每个具体的操作都有对应的stage
```go
func subtractStage(left interface{}, right interface{}, parameters Parameters) (interface{}, error) {
    return left.(float64) - right.(float64), nil
}
func multiplyStage(left interface{}, right interface{}, parameters Parameters) (interface{}, error) {
    return left.(float64) * right.(float64), nil
}
```
它们被放到一个map表里
```go
var stageSymbolMap = map[OperatorSymbol]evaluationOperator{
    EQ:             equalStage,
    NEQ:            notEqualStage,
    GT:             gtStage,
    LT:             ltStage,
    GTE:            gteStage,
    LTE:            lteStage,
    REQ:            regexStage,
    NREQ:           notRegexStage,
    AND:            andStage,
    OR:             orStage,
    IN:             inStage,
    BITWISE_OR:     bitwiseOrStage,
    BITWISE_AND:    bitwiseAndStage,
    BITWISE_XOR:    bitwiseXORStage,
    BITWISE_LSHIFT: leftShiftStage,
    BITWISE_RSHIFT: rightShiftStage,
    PLUS:           addStage,
    MINUS:          subtractStage,
    MULTIPLY:       multiplyStage,
    DIVIDE:         divideStage,
    MODULUS:        modulusStage,
    EXPONENT:       exponentStage,
    NEGATE:         negateStage,
    INVERT:         invertStage,
    BITWISE_NOT:    bitwiseNotStage,
    TERNARY_TRUE:   ternaryIfStage,
    TERNARY_FALSE:  ternaryElseStage,
    COALESCE:       ternaryElseStage,
    SEPARATE:       separatorStage,
}
```

## 执行阶段
```go
/*
    Same as `Eval`, but automatically wraps a map of parameters into a `govalute.Parameters` structure.
*/
func (this EvaluableExpression) Evaluate(parameters map[string]interface{}) (interface{}, error) {

    if parameters == nil {
        return this.Eval(nil)
    }

    /*
    Runs the entire expression using the given [parameters].
    e.g., If the expression contains a reference to the variable "foo", it will be taken from `parameters.Get("foo")`.

    This function returns errors if the combination of expression and parameters cannot be run,
    such as if a variable in the expression is not present in [parameters].

    In all non-error circumstances, this returns the single value result of the expression and parameters given.
    e.g., if the expression is "1 + 1", this will return 2.0.
    e.g., if the expression is "foo + 1" and parameters contains "foo" = 2, this will return 3.0
    */
    return this.Eval(MapParameters(parameters)) {
        return this.evaluateStage(this.evaluationStages, parameters) {
            //这里其实对应的是这个stage, 比如是multiplyStage, 执行
            //left.(float64) * right.(float64), nil
            return stage.operator(left, right, parameters)
        }
    }
}
```

## 总结
这个库是个非常简化版的解释器. 也有标准的三步:
* token阶段, 对应ast
* planStage阶段, 每个基础操作都有个operaor, 对应执行树
* 执行阶段, 执行预定义好的动作operaor.