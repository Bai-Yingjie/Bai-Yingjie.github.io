- [abs代码细看](#abs代码细看)
  - [为什么stdin能被迭代?](#为什么stdin能被迭代)
  - [evalIdentifier()](#evalidentifier)
  - [回到for in](#回到for-in)
  - [现在清楚了](#现在清楚了)
- [abs代码](#abs代码)
  - [代码组织](#代码组织)
  - [代码风格](#代码风格)
  - [从cli开始](#从cli开始)
      - [全局env](#全局env)
      - [NewEnvironment](#newenvironment)
      - [基础对象](#基础对象)
      - [string基础对象](#string基础对象)
      - [Array](#array)
    - [内置函数](#内置函数)
    - [内置函数的实现](#内置函数的实现)
    - [内置函数如何被调用 -- Eval()的魔法](#内置函数如何被调用----eval的魔法)
  - [Run()](#run)
    - [require和source](#require和source)
  - [第一步 lexer](#第一步-lexer)
  - [第二步 parser](#第二步-parser)
    - [举例: ParseNumberLiteral](#举例-parsenumberliteral)
  - [第三步 ParseProgram](#第三步-parseprogram)
    - [parseReturnStatement](#parsereturnstatement)
    - [parseAssignStatement()](#parseassignstatement)
    - [parseExpressionStatement()](#parseexpressionstatement)
    - [以上三个函数, 都调用了parseExpression()](#以上三个函数-都调用了parseexpression)
  - [Eval()](#eval)
    - [ast.Node是不是树?](#astnode是不是树)
    - [树分叉一般发生在Program和BlockStatement级别](#树分叉一般发生在program和blockstatement级别)
    - [function](#function)
    - [Eval的实现](#eval的实现)
  - [ast](#ast)
  - [总结](#总结)
- [abs](#abs)
  - [abs -- go实现的类shell](#abs----go实现的类shell)
  - [字符串](#字符串)
  - [内置通用array结构](#内置通用array结构)
  - [内置通用hash结构, 不同于go的map, 这里的key只能是string](#内置通用hash结构-不同于go的map-这里的key只能是string)
  - [函数](#函数)
  - [内置函数](#内置函数-1)
  - [装饰器](#装饰器)
  - [自带的标准库](#自带的标准库)
    - [cli](#cli)
      - [cli举例](#cli举例)
      - [repl cli 举例](#repl-cli-举例)
    - [函数cache](#函数cache)

# abs代码细看
## 为什么stdin能被迭代?
官方文档中, `stdin()`是个函数, 可以用来从标准输入读取输入
```shell
echo("What do you like?")
echo("Oh, you like %s!", stdin()) # This line will block until user enters some text
```
但stdin又能用于for
```shell
# Will read all input to the
# stdin and output it back
for input in stdin {
    echo(input)
}

# Or from the REPL:

⧐  for input in stdin { echo((input.int() / 2).str() + "...try again:")  }
10
5...try again:
5
2.5...try again:

...
```
那么`stdin`到底是什么呢?
注意到`functions.go`里面, 注册内置函数的时候, 有:
```go
        // stdin()
        "stdin": &object.Builtin{
            Next:  stdinNextFn,
            Types: []string{},
            Fn:    stdinFn,
        },
```
是把内置对象放到`map[string]*object.Builtin`中, 它是全局的变量`evaluator.Fns`
object.Builtin是个结构体:
```go
type Builtin struct {
    Token    token.Token
    Fn       BuiltinFunction
    Next     func() (Object, Object)
    Types    []string
    Iterable bool
}
```
stdin有点特殊, 它除了有Fn函数, 还有Next函数. 在所有内置的方法中, 只有stdin有Next函数.
```go
// stdin() -- implemented with 2 functions
func stdinFn(tok token.Token, env *object.Environment, args ...object.Object) object.Object {
    v := scanner.Scan()

    if !v {
        return EOF
    }

    return &object.String{Token: tok, Value: scanner.Text()}
}
func stdinNextFn() (object.Object, object.Object) {
    v := scanner.Scan()

    if !v {
        return nil, EOF
    }

    defer func() {
        scannerPosition += 1
    }()
    return &object.Number{Value: float64(scannerPosition)}, &object.String{Token: tok, Value: scanner.Text()}
}
```

## evalIdentifier()
这个函数中, 先是在env变量里面找, 优先返回变量; 其次返回builtin的对象.  
比如node.Value是"stdin"的时候, 就返回上面注册的stdin的对象
```go
func evalIdentifier(
    node *ast.Identifier,
    env *object.Environment,
) object.Object {
    if val, ok := env.Get(node.Value); ok {
        return val
    }

    if builtin, ok := Fns[node.Value]; ok {
        return builtin
    }

    return newError(node.Token, "identifier not found: "+node.Value)
}
```
* env是管变量对象的
* Fns是管内建对象的

## 回到for in
for in结构的执行在Eval()函数里
```go
    case *ast.ForInExpression:
        return evalForInExpression(node, env)
    case *ast.Identifier:
        return evalIdentifier(node, env)
```
evalForInExpression()函数中, 先调用Eval()得到iterable, 如果这个iterable是`*object.Builtin`类型, 
```go
// for k,v in 1..10 {v}
func evalForInExpression(
    fie *ast.ForInExpression,
    env *object.Environment,
) object.Object {
    iterable := Eval(fie.Iterable, env)
    // If "k" and "v" were already declared, let's keep
    // them aside...
    existingKeyIdentifier, okk := env.Get(fie.Key)
    existingValueIdentifier, okv := env.Get(fie.Value)

    // ...so that we can restore them after the for
    // loop is over
    //这个defer用的漂亮: 后面的loopIterable()函数会给fie.Key和fie.Value赋新值
    //这里用defer保证退出的时候, 恢复原Key Value的值, 或者如果当前env就没有这两个变量, 就删掉后面生成的.
    //漂亮!!!!! 事情还没做, 就把清理工作写好了!
    defer func() {
        if okk {
            env.Set(fie.Key, existingKeyIdentifier)
        } else {
            env.Delete(fie.Key)
        }

        if okv {
            env.Set(fie.Value, existingValueIdentifier)
        } else {
            env.Delete(fie.Value)
        }
    }()

    switch i := iterable.(type) {
    case object.Iterable:
        defer func() {
            i.Reset()
        }()

        return loopIterable(i.Next, env, fie, 0)
    case *object.Builtin:
        //这里正好对应"stdin"是有Next函数的
        if i.Next == nil {
            return newError(fie.Token, "builtin function cannot be used in loop")
        }

        return loopIterable(i.Next, env, fie, 0)
    default:
        return newError(fie.Token, "'%s' is a %s, not an iterable, cannot be used in for loop", i.Inspect(), i.Type())
    }
}
```
loopIterable()函数要求迭代函数next每次调用都返回一个k, v对. 这也是为什么next函数的签名必须是`func() (object.Object, object.Object)`. 这个k, v对会当作变量保存在env中.
```go
// This function iterates over an iterable
// represented by the next() function: everytime
// we call it, a new kv pair is popped from the
// iterable
func loopIterable(next func() (object.Object, object.Object), env *object.Environment, fie *ast.ForInExpression, index int64) object.Object {
    // Let's get the first kv pair out
    k, v := next()

    // Let's keep going until there are no
    // more kv pairs
    for k != nil && v != EOF {
        // set the special k v variables in the
        // environment
        env.Set(fie.Key, k)
        env.Set(fie.Value, v)
        res := Eval(fie.Block, env)

        if isError(res) {
            // If we have an error it could be:
            // * a break, so we get out of the loop
            // * a continue, so we go ahead with the next execution
            // * an actual error, so we wreak havoc
            switch res.(type) {
            case *object.BreakError:
                return NULL
            case *object.ContinueError:

            case *object.Error:
                return res
            }
        }

        // We had a return from within the FOR..IN loop
        switch res.(type) {
        case *object.ReturnValue:
            return res
        default:
            // do nothing
        }

        // Let's increment our index, and
        // pull the next kv pair
        index++
        k, v = next()
    }

    if k == nil || v == EOF {
        // If the index we're at is 0, it means the iterable
        // was empty. If so, let's try to eval its else condition
        // (eg. for x in [] {...} else {...})
        if index == 0 && fie.Alternative != nil {
            return Eval(fie.Alternative, env)
        }
    }

    return NULL
}
```

## 现在清楚了
* stdin()是个内置函数
* 而stdin本身是个关键词, 内部表达是个Builtin的对象, 这个对象有Next方法, 能够被迭代
* 所有内建函数的名字都是Builtin对象. abs应该是允许变量名和内置对象名重名的, 这种情况下, 用户的变量名优先.

# abs代码
## 代码组织
* object: object是abs对象的抽象, 是个interface. 实现了Type() Inspect() Json()这三个基本函数的都是object对象. abs对象是abs语法里面的string array hash等基础数据类型. 
environment.go 是对函数上下文的抽象, 类似frame的概念. 其核心是per function的env, 里面用`store map[string]Object` 表示变量
* evaluator: 核心是递归调用的Eval()函数, 里面有对语法的底层执行代码
functions.go里面实现了基础数据类型的内置方法, 比如len()函数;
* lexer: lexer把原始的string输入, 通过nextToken()和peekToken等函数的向后移动, 来遍历原始输入.
* parser: parser.go提供主要的词法分析. 有优先级的定义; 随着token的向后移动, 目的是生成对每个词法的`ast.Statement`, 即ast树上的node
```go
const (
    _ int = iota
    LOWEST
    AND         // && or ||
    EQUALS      // == or !=
    LESSGREATER // > or <
    SUM         // + or -
    PRODUCT     // * or / or ^
    RANGE       // ..
    PREFIX      // -X or !X
    CALL        // myFunction(X)
    INDEX       // array[index]
    QUESTION    // some?.function() or some?.property
    DOT         // some.function() or some.property
)
```
* ast: ast的Node是个树, 但在表现上是以interface存在的. node扩展成statement和expression
* token: token.go很简单, 定义了token的常量. 就像C里面的宏定义. 比如
`PLUS = "+"`作用是如果改语法的token关键词, 改这个文件就好了
* util:

## 代码风格
代码里面用了大量的map数据结构, map表意清楚, 用法直观, 深受喜爱.
比如. 解释语言中, 什么东西都是从string解释而来, nubmer也不意外.
abs支持numberLiteral(即string化的number)后面带k m b等单位
```go
// NumberAbbreviations is a list of abbreviations that can be used in numbers eg. 1k, 20B
var NumberAbbreviations = map[string]float64{
    "k": 1000,
    "m": 1000000,
    "b": 1000000000,
    "t": 1000000000000,
}
```
在parser解析number的时候, 这个number是个数字的字符串, 比如1 1.1 或者1.1k
下面的代码就是看最后一个字符是否带量级标记, 是的话就查map表得到`abbr`的数字.
abs的数字默认是float64类型.
```go
func (p *Parser) ParseNumberLiteral() ast.Expression {
    lit := &ast.NumberLiteral{Token: p.curToken}
    var abbr float64
    var ok bool
    number := p.curToken.Literal

    // Check if the last character of this number is an abbreviation
    if abbr, ok = token.NumberAbbreviations[strings.ToLower(string(number[len(number)-1]))]; ok {
        number = p.curToken.Literal[:len(p.curToken.Literal)-1]
    }
    
    value, err := strconv.ParseFloat(number, 64)
    if err != nil {
        msg := fmt.Sprintf("could not parse %q as number", number)
        p.reportError(msg, p.curToken)
        return nil
    }
    //到这里做的工作就是把1k转为1 * 1000
    if abbr != 0 {
        value *= abbr
    }
    //这里的value已经是float64类型了
    lit.Value = value

    return lit
}
```

## 从cli开始
go build就能编译出abs
```go
// 这个函数支持交互式, 也支持执行脚本, 通过命令参数区分
func BeginRepl(args []string, version string) {
    //比如是脚本模式
    //这里的set就是e.store[name] = val; e是下面的*object.Environment
    env.Set("ABS_INTERACTIVE", evaluator.FALSE)
    code, err := ioutil.ReadFile(args[1])
    Run(string(code), false)
}
```
#### 全局env
~~每个function都有一个env~~
基本上, 一般都只有一个env. 这是类shell解释器的通常思路: 所有变量, 函数等都是全局的. 一个文件就像一个大函数.
```go
//这里的env是指interp包的全局的环境, 包括所有变量等
var env *object.Environment

// Environment represent the environment associated
// with the execution context of an ABS script: it
// holds all variables etc.
type Environment struct {
    store map[string]Object
    // Arguments this environment was created in.
    // When we call function(1, 2, 3), a new environment
    // for the function to execute is created, and 1/2/3
    // are recorded as arguments for this environment.
    //
    // Later, if we need to access the arguments passed
    // to the function, we can refer back to them
    // through env.CurrentArgs. This is how ... is
    // implemented.
    CurrentArgs []Object
    //每个function都有一个env, 这里的outer是其上层env
    outer       *Environment
    // Used to capture output. This is typically os.Stdout,
    // but you could capture in any io.Writer of choice
    Writer io.Writer
    // Dir represents the directory from which we're executing code.
    // It starts as the directory from which we invoke the ABS
    // executable, but changes when we call require("...") as each
    // require call resets the dir to its own directory, so that
    // relative imports work.
    //
    // If we have script A and B in /tmp, A can require("B")
    // wihout having to specify its full absolute path
    // eg. require("/tmp/B")
    Dir string
    // Version of the ABS runtime
    Version string
}

```

#### NewEnvironment
按照shell的惯例, 所有变量都应该是一个"frame"
* 在repl的init里会新建env
* 在require("file.abs")的时候会新建env
* "{}".json()方法里面会新建env.

特别的, 执行用户自定义的function时(`applyFunction()函数`)也会新建一个env, 但这个env是`NewEnclosedEnvironment()`创建的, 它能够访问外面的(outer)env
```go
// NewEnclosedEnvironment creates an environment
// with another one embedded to it, so that the
// new environment has access to identifiers stored
// in the outer one.
func NewEnclosedEnvironment(outer *Environment, args []Object) *Environment {
    env := NewEnvironment(outer.Writer, outer.Dir, outer.Version)
    env.outer = outer
    env.CurrentArgs = args
    return env
}
```

#### 基础对象
abs种的基础对象有:
```go
const (
    NULL_OBJ  = "NULL"
    ERROR_OBJ = "ERROR"

    NUMBER_OBJ  = "NUMBER"
    BOOLEAN_OBJ = "BOOLEAN"
    STRING_OBJ  = "STRING"

    RETURN_VALUE_OBJ = "RETURN_VALUE"

    // ANY_OBJ represents any ABS type
    ANY_OBJ      = "ANY"
    FUNCTION_OBJ = "FUNCTION"
    BUILTIN_OBJ  = "BUILTIN"

    ARRAY_OBJ = "ARRAY"
    HASH_OBJ  = "HASH"
)
```
```go
// 对象是个interface抽象
type Object interface {
    Type() ObjectType
    Inspect() string
    Json() string
}
```
比如对Number的抽象就是
```go
type Number struct {
    Token token.Token
    Value float64
}

func (n *Number) Type() ObjectType { return NUMBER_OBJ }

// If the number we're dealing with is
// an integer, print it as such (1.0000 becomes 1).
// If it's a float, let's remove as many zeroes
// as possible (1.10000 becomes 1.1).
func (n *Number) Inspect() string {
    if n.IsInt() {
        return fmt.Sprintf("%d", int64(n.Value))
    }
    return strconv.FormatFloat(n.Value, 'f', -1, 64)
}
func (n *Number) IsInt() bool {
    return n.Value == float64(int64(n.Value))
}
func (n *Number) Json() string       { return n.Inspect() }
func (n *Number) ZeroValue() float64 { return float64(0) }
func (n *Number) Int() int           { return int(n.Value) }
```

大部分基础对象有Value, 小部分没有
```go
//Boolean就有Value, 很显然, 是个bool
type Boolean struct {
    Token token.Token
    Value bool
}

//Function没有Value, 但也实现了Object的方法, 也是Object对象
type Function struct {
    Token      token.Token
    Name       string
    Parameters []*ast.Parameter
    Body       *ast.BlockStatement
    Env        *Environment
    Node       *ast.FunctionLiteral
}
```

#### string基础对象
string有点特别. 为了支持直接调用shell命令, 字符串还要实现Ok和Done方法
string的典型用法:
```go
// cmd = `ls -la`
// type(cmd) // STRING
// cmd.ok // TRUE
//
// cmd = `curlzzzzz`
// type(cmd) // STRING
// cmd.ok // FALSE
//
// cmd = `sleep 10 &`
// type(cmd) // STRING
// cmd.done // FALSE
// cmd.wait() // ...
// cmd.done // TRUE
```
所以string有更多的属性
```go
type String struct {
    Token  token.Token
    Value  string
    Ok     *Boolean  // A special property to check whether a command exited correctly
    Cmd    *exec.Cmd // A special property to access the underlying command
    Stdout *bytes.Buffer
    Stderr *bytes.Buffer
    Done   *Boolean
    mux    *sync.Mutex
}

// string的几个函数写的漂亮
func (s *String) Type() ObjectType  { return STRING_OBJ }
func (s *String) Inspect() string   { return s.Value }
func (s *String) Json() string      { return `"` + strings.ReplaceAll(s.Inspect(), `"`, `\"`) + `"` }
func (s *String) ZeroValue() string { return "" }
func (s *String) HashKey() HashKey {
    return HashKey{Type: s.Type(), Value: s.Value}
}

// Function that ensure a mutex
// instance is created on the
// string
func (s *String) mustHaveMutex() {
    if s.mux == nil {
        s.mux = &sync.Mutex{}
    }
}

// To be called when the command
// is done. Releases the internal
// mutex.
func (s *String) SetDone() {
    s.mustHaveMutex()
    s.mux.Unlock()
}

// To be called when the command
// is starting in background, so
// that anyone accessing it will
// be blocked.
func (s *String) SetRunning() {
    s.mustHaveMutex()
    s.mux.Lock()
}

// To be called when we want to
// wait on the background command
// to be done.
func (s *String) Wait() {
    s.mustHaveMutex()
    s.mux.Lock()
    s.mux.Unlock()
}

// To be called when we want to
// kill the background command
func (s *String) Kill() error {
    err := s.Cmd.Process.Kill()

    // The command value includes output and possible error
    // We might want to change this
    output := s.Stdout.String()
    outputErr := s.Stderr.String()
    s.Value = strings.TrimSpace(output) + strings.TrimSpace(outputErr)

    if err != nil {
        return err
    }

    s.Done = TRUE
    return nil
}

// Sets the result of the underlying command
// on the string.
// 3 things are set:
// - the string itself (output of the command)
// - str.ok
// - str.done
func (s *String) SetCmdResult(Ok *Boolean) {
    s.Ok = Ok
    var output string

    if Ok.Value {
        output = s.Stdout.String()
    } else {
        output = s.Stderr.String()
    }

    // trim space at both ends of out.String(); works in both linux and windows
    s.Value = strings.TrimSpace(output)
    s.Done = TRUE
}
```

#### Array
abs的Array是个万能容器
```go
type Array struct {
    Token    token.Token
    Elements []Object
    // ... is aliased to an array of arguments.
    //
    // Since this is a special case of an array,
    // we need a flag to make sure we know when
    // to unpack them, else if we do func(...),
    // func would receive only one array argument
    // as opposd to the unpacked arguments.
    IsCurrentArgs bool
    position      int
}
```

### 内置函数
`evaluator/evaluator.go`中, 有内置的函数. 内置函数放在一个全局的map表中
```go
var (
    NULL  = object.NULL
    EOF   = object.EOF
    TRUE  = object.TRUE
    FALSE = object.FALSE
    Fns   map[string]*object.Builtin
)

func init() {
    Fns = getFns()
}

这个map里面是: 部分例子, 很多都是针对string的.
    map[string]*object.Builtin{
        // len(var:"hello")
        "len": &object.Builtin{
            Types: []string{object.STRING_OBJ, object.ARRAY_OBJ},
            Fn:    lenFn,
        },
        // cd() or cd(path)
        "cd": &object.Builtin{
            Types: []string{},
            Fn:    cdFn,
        },
        // echo(arg:"hello")
        "echo": &object.Builtin{
            Types: []string{},
            Fn:    echoFn,
        },
        // int(string:"123")
        // int(number:"123")
        "int": &object.Builtin{
            Types: []string{object.STRING_OBJ, object.NUMBER_OBJ},
            Fn:    intFn,
        },
        // round(string:"123.1")
        // round(number:"123.1", 2)
        "round": &object.Builtin{
            Types: []string{object.STRING_OBJ, object.NUMBER_OBJ},
            Fn:    roundFn,
        },
        // number(string:"1.23456")
        "number": &object.Builtin{
            Types: []string{object.STRING_OBJ, object.NUMBER_OBJ},
            Fn:    numberFn,
        },
        // stdin()
        "stdin": &object.Builtin{
            Next:  stdinNextFn,
            Types: []string{},
            Fn:    stdinFn,
        },
        // env(variable:"PWD") or env(string:"KEY", string:"VAL")
        "env": &object.Builtin{
            Types: []string{},
            Fn:    envFn,
        },
        // type(variable:"hello")
        "type": &object.Builtin{
            Types: []string{},
            Fn:    typeFn,
        },
        // fn.call(args_array)
        "call": &object.Builtin{
            Types: []string{object.FUNCTION_OBJ, object.BUILTIN_OBJ},
            Fn:    callFn,
        },
        // "{}".json()
        // Converts a valid JSON document to an ABS hash.
        "json": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    jsonFn,
        },
        // "a %s".fmt(b)
        "fmt": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    fmtFn,
        },
        // sum(array:[1, 2, 3])
        "sum": &object.Builtin{
            Types: []string{object.ARRAY_OBJ},
            Fn:    sumFn,
        },
        // sort(array:[1, 2, 3])
        "sort": &object.Builtin{
            Types: []string{object.ARRAY_OBJ},
            Fn:    sortFn,
        },
        // map(array:[1, 2, 3], function:f(x) { x + 1 })
        "map": &object.Builtin{
            Types: []string{object.ARRAY_OBJ},
            Fn:    mapFn,
        },
        // every(array:[1, 2, 3], function:f(x) { x == 2 })
        "every": &object.Builtin{
            Types: []string{object.ARRAY_OBJ},
            Fn:    everyFn,
        },
        // find(array:[1, 2, 3], function:f(x) { x == 2 })
        "find": &object.Builtin{
            Types: []string{object.ARRAY_OBJ},
            Fn:    findFn,
        },
        // repeat("abc", 3)
        "repeat": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    repeatFn,
        },
        // sleep(3000)
        "sleep": &object.Builtin{
            Types: []string{object.NUMBER_OBJ},
            Fn:    sleepFn,
        },
        // source("file.abs") -- soure a file, with access to the global environment
        "source": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    sourceFn,
        },
        // require("file.abs") -- require a file without giving it access to the global environment
        "require": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    requireFn,
        },
        // exec(command) -- execute command with interactive stdIO
        "exec": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    execFn,
        },
        // eval(code) -- evaluates code in the context of the current ABS environment
        "eval": &object.Builtin{
            Types: []string{object.STRING_OBJ},
            Fn:    evalFn,
        },
}

type Builtin struct {
    Token    token.Token
    Fn       BuiltinFunction
    Next     func() (Object, Object)
    Types    []string //这个函数支持的对象类型, 可以支持多个类型, 这里是个切片
    Iterable bool
}
```

### 内置函数的实现
比如, 内置对象是object.Builtin类型
```go
type Builtin struct {
    Token    token.Token
    Fn       BuiltinFunction
    Next     func() (Object, Object)
    Types    []string
    Iterable bool
}

//其中BuiltinFunction接受token, env, 其他变长的Object类型参数. 返回Object
type BuiltinFunction func(tok token.Token, env *Environment, args ...Object) Object
```
比如len("hello")的实现就是
```go
        // len(var:"hello")
        "len": &object.Builtin{
            Types: []string{object.STRING_OBJ, object.ARRAY_OBJ},
            Fn:    lenFn,
        },
```
这个Fn的实现是
```go
// len(var:"hello")
func lenFn(tok token.Token, env *object.Environment, args ...object.Object) object.Object {
    //args的个数应该是1
    //如果有多个参数, 第一个参数的类型要在[][]string[0]中, 第二个参数要在[][]string[1]中
    //这里只有一个arg, 它的类型要在{object.STRING_OBJ, object.ARRAY_OB}中
    err := validateArgs(tok, "len", args, 1, [][]string{{object.STRING_OBJ, object.ARRAY_OBJ}})
    if err != nil {
        return err
    }

    switch arg := args[0].(type) {
    case *object.Array:
        return &object.Number{Token: tok, Value: float64(len(arg.Elements))}
    case *object.String:
        return &object.Number{Token: tok, Value: float64(len(arg.Value))}
    default:
        return newError(tok, "argument to `len` not supported, got %s", args[0].Type())
    }
}
```

### 内置函数如何被调用 -- Eval()的魔法
内置函数的基本调用形式有2种
```shell
#函数式
len("nihaoma")
7

#对象方法方式
"nihaoma".len()
7

#连续调用
"nihaoma".len().type()
NUMBER

#连续的连续
"nihaoma".len().type().type()
STRING
```
调用到Fn的路径是, `evaluator/evaluator.go`中:
```go
func applyMethod(tok token.Token, o object.Object, me *ast.MethodExpression, env *object.Environment, args []object.Object) object.Object {
    method := me.Method.String()
    //hash类型可以有用户自定义的方法
    hash, isHash := o.(*object.Hash)
    //hash类型时,调用用户态自定义的函数
    // If so, run the user-defined function
    if isHash && hash.GetKeyType(method) == object.FUNCTION_OBJ {
        pair, _ := hash.GetPair(method)
        //自定义的函数就不传obj本身了. 是否可以改进成也传入对象本身? 这样这个函数就能访问对象的属性了.
        return applyFunction(tok, pair.Value.(*object.Function), env, args)
    }
    
    // Now, check if there is a builtin function with the given name
    f, ok := Fns[method]
    
    //检查参数是否类型合法
    if util.Contains(f.Types, string(o.Type()))
    
    //真正的黑魔法, 调用这个函数
    //第一个参数是对象本身, 其他的参数依次append到后面. 这里写的漂亮
    args = append([]object.Object{o}, args...)
    return f.Fn(tok, env, args...)
}

func applyFunction(tok token.Token, fn object.Object, env *object.Environment, args []object.Object) object.Object {
    switch fn := fn.(type) {
    case *object.Function:
        extendedEnv, err := extendFunctionEnv(fn, args)

        if err != nil {
            return err
        }
        evaluated := Eval(fn.Body, extendedEnv)
        return unwrapReturnValue(evaluated)

    case *object.Builtin:
        return fn.Fn(tok, env, args...)

    default:
        return newError(tok, "not a function: %s", fn.Type())
    }
}
```

上面的applyMethod被Eval()调用
evaluator.go里面的Eval是个核心函数, 它根据抽象语法树(ast)的node类型, 执行相应的方法. 实际上, Eval()不仅能执行内置的函数, 它能执行任意的"文本"代码. 
详见下文

## Run()
cli刚开始的时候, 调用Run()来解释执行代码
```go
// 这个函数支持交互式, 也支持执行脚本, 通过命令参数区分
func BeginRepl(args []string, version string) {
    //比如是脚本模式
    //这里的set就是e.store[name] = val; e是下面的*object.Environment
    env.Set("ABS_INTERACTIVE", evaluator.FALSE)
    code, err := ioutil.ReadFile(args[1])
    Run(string(code), false)
}
```
其中, Run()函数使用了lexer parser evaluator等包的功能, 负责解释执行string类型的code.

```go
func Run(code string, interactive bool) {
    //词法器
    lex := lexer.New(code)
    //解析器
    p := parser.New(lex)
    //现在是program了
    program := p.ParseProgram()
    
    evaluated := evaluator.BeginEval(program, env, lex)
}

// BeginEval (program, env, lexer) object.Object
// REPL and testing modules call this function to init the global lexer pointer for error location
// NB. Eval(node, env) is recursive
func BeginEval(program ast.Node, env *object.Environment, lexer *lexer.Lexer) object.Object {
    //这里竟然是属于evaluator的全局变量: 用于出错时定位错误在哪一行
    // global lexer
    lex = lexer
    // run the evaluator
    //这里的Eval和上面的就对上了
    return Eval(program, env)
}
```
Eval program实际上就是对program里的每个statement, 递归调用Eval(), 如果返回ReturnValue, 说明可以提前返回; 或者返回Error也要提前返回
```go
func evalProgram(program *ast.Program, env *object.Environment) object.Object {
    var result object.Object
    for _, statement := range program.Statements {
        result = Eval(statement, env)
        switch result := result.(type) {
        case *object.ReturnValue:
            return result.Value
        case *object.Error:
            return result
        }
    }

    return result
}
```
### require和source
`evaluator/functions.go`里面的`doSource()`函数和Run的流程类似, 它读取一个source文件, 创建lexer, parser, 然后Eval这个program.

## 第一步 lexer
基本上, lexer的原理是按每个单个字符读入分析. lexer关注字符, 所以每个字符都是rune类型.
lexer关注字符的位置; 基本上是把包含code的string, 转化为`input []rune`, 和其他一些辅助属性. 用于后续解析.
```go
type Lexer struct {
    position     int  // current position in input (points to current char)
    readPosition int  // current reading position in input (after current char)
    ch           rune // current rune under examination
    input        []rune
    // map of input line boundaries used by linePosition() for error location
    lineMap [][2]int // array of [begin, end] pairs: [[0,12], [13,22], [23,33] ... ]
}
```
Run()的第一步就是`lex := lexer.New(code)`
```go
func New(in string) *Lexer {
    l := &Lexer{input: []rune(in)}
    // map the input line boundaries for CurrentLine()
    l.buildLineMap()
    // read the first char
    l.readChar()
    return l
}
```

## 第二步 parser
parser是从原始文法, 到ast树的关键.  
ast token等基础概念可以interface抽象, 而parser是要实际干活的了, 它是个struct.  
Parser持有一个Lexer的实例, 当前token, 下个token; 还有index表达, 属性表达.  
另外还有基础词法parse的函数集.
```go
type Parser struct {
    l      *lexer.Lexer
    errors []string

    curToken  token.Token
    peekToken token.Token

    // support assignment to index expressions: a[0] = 1, h["a"] = 1
    prevIndexExpression *ast.IndexExpression

    // support assignment to hash property h.a = 1
    prevPropertyExpression *ast.PropertyExpression

    //如果都是预定义好的解析函数, 直接用全局变量函数不是更好?
    prefixParseFns map[token.TokenType]prefixParseFn
    infixParseFns  map[token.TokenType]infixParseFn
}
```
两个register函数注册解析函数到parser. 
```go
func (p *Parser) registerPrefix(tokenType token.TokenType, fn prefixParseFn) {
    p.prefixParseFns[tokenType] = fn
}

func (p *Parser) registerInfix(tokenType token.TokenType, fn infixParseFn) {
    p.infixParseFns[tokenType] = fn
}
```
New()方法会注册基础关键词的解析函数
```go
func New(l *lexer.Lexer) *Parser {
    p := &Parser{
        l:      l,
        errors: []string{},
    }
    
    //每个关键词, 都对应一个预定义的解析函数. 比如token.TRUE对一个ParseBoolean.
    p.prefixParseFns = make(map[token.TokenType]prefixParseFn)
    p.registerPrefix(token.IDENT, p.parseIdentifier)
    //前文中, 分析过ParseNumberLiteral, 是把string的"1k"转为float64类型的值
    p.registerPrefix(token.NUMBER, p.ParseNumberLiteral)
    p.registerPrefix(token.STRING, p.ParseStringLiteral)
    p.registerPrefix(token.NULL, p.ParseNullLiteral)
    p.registerPrefix(token.BANG, p.parsePrefixExpression)
    p.registerPrefix(token.MINUS, p.parsePrefixExpression)
    p.registerPrefix(token.TILDE, p.parsePrefixExpression)
    p.registerPrefix(token.TRUE, p.ParseBoolean)
    p.registerPrefix(token.FALSE, p.ParseBoolean)
    
    ...
    //infix又是什么呢?
    p.infixParseFns = make(map[token.TokenType]infixParseFn)
    p.registerInfix(token.QUESTION, p.parseQuestionExpression)
    p.registerInfix(token.DOT, p.parseDottedExpression)
    p.registerInfix(token.PLUS, p.parseInfixExpression)
    p.registerInfix(token.MINUS, p.parseInfixExpression)
    p.registerInfix(token.SLASH, p.parseInfixExpression)
    p.registerInfix(token.EXPONENT, p.parseInfixExpression)
    p.registerInfix(token.MODULO, p.parseInfixExpression)
    
    ...
    
    // 最后读两个token出来, 让事情转起来.
    // Read two tokens, so curToken and peekToken are both set
    p.nextToken()
    p.nextToken()

    return p
}
```
prefix和infix对应的是关键词前 关键词中需要的函数; 区别是infix需要额外的入参. 可能就是前序解析出来的ast.Expression
```go
type (
    prefixParseFn func() ast.Expression
    infixParseFn  func(ast.Expression) ast.Expression
)

```

### 举例: ParseNumberLiteral
在parser解析number的时候, 这个number是个数字的字符串, 比如1 1.1 或者1.1k  
下面的代码就是看最后一个字符是否带量级标记, 是的话就查map表得到`abbr`的数字.  
abs的数字默认是float64类型.
```go
func (p *Parser) ParseNumberLiteral() ast.Expression {
    lit := &ast.NumberLiteral{Token: p.curToken}
    var abbr float64
    var ok bool
    number := p.curToken.Literal

    // Check if the last character of this number is an abbreviation
    if abbr, ok = token.NumberAbbreviations[strings.ToLower(string(number[len(number)-1]))]; ok {
        number = p.curToken.Literal[:len(p.curToken.Literal)-1]
    }
    
    value, err := strconv.ParseFloat(number, 64)
    if err != nil {
        msg := fmt.Sprintf("could not parse %q as number", number)
        p.reportError(msg, p.curToken)
        return nil
    }
    //到这里做的工作就是把1k转为1 * 1000
    if abbr != 0 {
        value *= abbr
    }
    //这里的value已经是float64类型了
    lit.Value = value

    return lit
}
```

## 第三步 ParseProgram
parser现在已经有lexer实例, 从而有了从原始string读token的能力; 也注册了基本词法的解析器.  
现在perser可以把token转换为ast.Expression  
后面会看到, `p.nextToken()`会随着每个词法的解析, 根据情况来向后移动.
```go
program := p.ParseProgram()
```
这个ParseProgram()函数也写的非常漂亮
```go
func (p *Parser) ParseProgram() *ast.Program {
    program := &ast.Program{}
    program.Statements = []ast.Statement{}

    //从头到尾逐个token来解析
    for !p.curTokenIs(token.EOF) {
        //解析完成后放到program.Statements中
        stmt := p.parseStatement()
        if stmt != nil {
            program.Statements = append(program.Statements, stmt)
        }
        //下一个token, 实际是调用底层的lexer的 p.l.NextToken()
        p.nextToken()
    }

    return program
}
```
parseStatement()先检查是否是Return类型的, 然后尝试先按Assign类型去解析, 最后用Expression类型去解析. 
```go
func (p *Parser) parseStatement() ast.Statement {
    if p.curToken.Type == token.RETURN {
        return p.parseReturnStatement()
    }

    statement := p.parseAssignStatement()
    if statement != nil {
        return statement
    }

    return p.parseExpressionStatement()
}
```

### parseReturnStatement
题外: 这个peekToken的设计真是绝了. 让我想起了awk的getline()内置函数. `peek`这个词用的真是绝了.

```go
// return x
func (p *Parser) parseReturnStatement() *ast.ReturnStatement {
    //这里的curToken就是"return"
    stmt := &ast.ReturnStatement{Token: p.curToken}
    returnToken := p.curToken

    // return;
    if p.peekTokenIs(token.SEMICOLON) {
        stmt.ReturnValue = &ast.NullLiteral{Token: p.curToken}
    } else if p.peekTokenIs(token.RBRACE) || p.peekTokenIs(token.EOF) {
        // return
        stmt.ReturnValue = &ast.NullLiteral{Token: returnToken}
    } else {
        // return xyz
        p.nextToken()
        //LOWEST的意思是当前的优先级最低. parseExpression里面的token现在还不知道, 每个token都有它的优先级的.
        stmt.ReturnValue = p.parseExpression(LOWEST)
    }

    if p.peekTokenIs(token.SEMICOLON) {
        p.nextToken()
    }

    return stmt
}
```

### parseAssignStatement()
这个函数处理各种赋值语句.
```go
// assign to variable: x = y
// destructuring assignment: x, y = [z, zz]
// assign to index expressions: a[0] = 1, h["a"] = 1
// assign to hash property expressions: h.a = 1
func (p *Parser) parseAssignStatement() ast.Statement {
    stmt := &ast.AssignStatement{}

    // Is this a regular x = y assignment?
    if p.peekTokenIs(token.COMMA) {
        lexerPosition := p.l.CurrentPosition()
        // Let's figure out if we are destructuring x, y = [z, zz]
        if !p.curTokenIs(token.IDENT) {
            return nil
        }

        stmt.Names = p.parseDestructuringIdentifiers()

        if !p.peekTokenIs(token.ASSIGN) {
            p.Rewind(lexerPosition)
            return nil
        }
    } else if p.curTokenIs(token.IDENT) {
        stmt.Name = &ast.Identifier{Token: p.curToken, Value: p.curToken.Literal}
    } else if p.curTokenIs(token.ASSIGN) {
        stmt.Token = p.curToken
        if p.prevIndexExpression != nil {
            // support assignment to indexed expressions: a[0] = 1, h["a"] = 1
            stmt.Index = p.prevIndexExpression
            p.nextToken()
            stmt.Value = p.parseExpression(LOWEST)
            // consume the IndexExpression
            p.prevIndexExpression = nil

            if p.peekTokenIs(token.SEMICOLON) {
                p.nextToken()
            }

            return stmt
        }
        if p.prevPropertyExpression != nil {
            // support assignment to hash properties: h.a = 1
            stmt.Property = p.prevPropertyExpression
            p.nextToken()
            stmt.Value = p.parseExpression(LOWEST)
            // consume the PropertyExpression
            p.prevPropertyExpression = nil

            if p.peekTokenIs(token.SEMICOLON) {
                p.nextToken()
            }

            return stmt
        }
    }

    if !p.peekTokenIs(token.ASSIGN) {
        return nil
    }

    p.nextToken()
    stmt.Token = p.curToken
    p.nextToken()

    stmt.Value = p.parseExpression(LOWEST)

    if p.peekTokenIs(token.SEMICOLON) {
        p.nextToken()
    }

    return stmt
}
```

### parseExpressionStatement()
```go
// (x * y) + z
func (p *Parser) parseExpressionStatement() *ast.ExpressionStatement {
    stmt := &ast.ExpressionStatement{Token: p.curToken}

    stmt.Expression = p.parseExpression(LOWEST)

    if p.peekTokenIs(token.SEMICOLON) {
        p.nextToken()
    }

    return stmt
}
```

### 以上三个函数, 都调用了parseExpression()
这个函数写的真好.
它调用了New parser时注册的各个词法的prefixParseFns和infixParseFns, 这是处理语法中最复杂的地方.
别看这个函数不长, 但却是prefix和infix等基础函数的总调用入口.
```go
func (p *Parser) parseExpression(precedence int) ast.Expression {
    prefix := p.prefixParseFns[p.curToken.Type]
    if prefix == nil {
        p.noPrefixParseFnError(p.curToken)
        return nil
    }
    leftExp := prefix()

    for !p.peekTokenIs(token.SEMICOLON) && precedence < p.peekPrecedence() {
        infix := p.infixParseFns[p.peekToken.Type]
        if infix == nil {
            return leftExp
        }

        p.nextToken()

        leftExp = infix(leftExp)
    }

    return leftExp
}
```

## Eval()
parser阶段, 把lexer的token按照字面语法移动, 得到相应的ast.  Expression, 这就是个ast.Node节点.  
node是个抽象, 实现了下面两个方法的实体类型, 都是node. 这个node就是ast的节点.  
evaluator.go里面的Eval是个核心函数, 它根据抽象语法树(ast)的node类型, 执行相应的方法. 实际上, Eval()不仅能执行内置的函数, 它能执行任意的"文本"代码. 

经过前面的三步, 每个词法都对应了一个ast.Node树, ast.Node是个抽象, 一般是Statement或Expression.

从表面上看, node只是个节点, 还是interface, 好像不是个树. 其定义如下:

```go
type Node interface {
    TokenLiteral() string
    String() string
}
```

特别的, Statement和Expression是Node的扩展抽象.
```go
// All statement nodes implement this
type Statement interface {
    Node
    statementNode()
}

// All expression nodes implement this
type Expression interface {
    Node
    expressionNode()
}
```
其中, Expression实现了statementNode()方法, 它本身是个Statement
```go
type ExpressionStatement struct {
    Token      token.Token // the first token of the expression
    Expression Expression
}

func (es *ExpressionStatement) statementNode()       {}
func (es *ExpressionStatement) TokenLiteral() string { return es.Token.Literal }
func (es *ExpressionStatement) String() string {
    if es.Expression != nil {
        return es.Expression.String()
    }
    return ""
}
```
### ast.Node是不是树?
表面上看, ast.Node跟树完全扯不上关系 -- 它只是个接口, 而且只有两个返回string的方法.

但其具体实现里, 都是有树的. 比如:
BlockStatement就包括了子节点Statements
```go
type BlockStatement struct {
    Token      token.Token // the { token
    Statements []Statement
}
```

一个Program也是Statements的集合, 属性里少了token字段, 但比BlockStatement概念要大, 是整个代码段的入口
```go
type Program struct {
    Statements []Statement
}
```

ExpressionStatement很常用:
```go
type ExpressionStatement struct {
    Token      token.Token // the first token of the expression
    Expression Expression
}
```

AssignStatement是个相对复杂的结构:
```go
type AssignStatement struct {
    Token    token.Token // the token.ASSIGN token
    Name     *Identifier
    Names    []Expression
    Index    *IndexExpression    // support assignment to indexed expressions: a[0] = 1, h["a"] = 1
    Property *PropertyExpression // support assignment to hash properties: h.a = 1
    Value    Expression
}
```

number是最简单的, 其值为翻译后的float64
```go
type NumberLiteral struct {
    Token token.Token
    Value float64
}
```

Boolean也是类似的
```go
type Boolean struct {
    Token token.Token
    Value bool
}
```

prefix是两个操作符, infix是三个(比如 x = a + b 中, infix指+)
```go
type PrefixExpression struct {
    Token    token.Token // The prefix token, e.g. !
    Operator string
    Right    Expression
}

type InfixExpression struct {
    Token    token.Token // The operator token, e.g. +
    Left     Expression
    Operator string
    Right    Expression
}
```

我最喜欢的for in组合
```go
type ForInExpression struct {
    Token       token.Token     // The 'for' token
    Block       *BlockStatement // The block executed inside the for loop
    Iterable    Expression      // An expression that should return an iterable ([1, 2, 3] or x in 1..10)
    Key         string
    Value       string
    Alternative *BlockStatement
}
```

### 树分叉一般发生在Program和BlockStatement级别
从上文可以看出, BlockStatement包括`Statements`集合, 一对`{...}`是个树, 里面的每个块也是个树; 块里面的Statements是顺序的.
这个是最自然的理解

### function
function地位特别, 但实现上也似乎没有专门对待. 有参数列表, 有函数体(是个 `* BlockStatement`类型), 就是个函数.
```go
type FunctionLiteral struct {
    Token      token.Token // The 'fn' token
    Name       string      // identifier for this function
    Parameters []*Parameter
    Body       *BlockStatement
}
```

### Eval的实现
Eval()函数我一行都没舍得删, 完全搬到这里来:
下面是已经实现了node抽象的实体类型
* `*ast.Program`
* `*ast.BlockStatement`
* `*ast.ExpressionStatement`
* `*ast.ReturnStatement`
* `*ast.AssignStatement`
* `*ast.NumberLiteral`
* `*ast.NullLiteral`
* `*ast.CurrentArgsLiteral`
* `*ast.StringLiteral`
* `*ast.Boolean`
* `*ast.PrefixExpression`
* `*ast.InfixExpression`
* `*ast.CompoundAssignment`
* `*ast.IfExpression`
* `*ast.WhileExpression`
* `*ast.ForExpression`
* `*ast.ForInExpression`
* `*ast.Identifier`
* `*ast.FunctionLiteral`
* `*ast.Decorator`
* `*ast.CallExpression`
* `*ast.MethodExpression`
* `*ast.PropertyExpression`
* `*ast.ArrayLiteral`
* `*ast.IndexExpression`
* `*ast.HashLiteral`
* `*ast.CommandExpression`
* `*ast.BreakStatement`
* `*ast.ContinueStatement`

Eval()的作用是根据输入的ast.Node和env, recursive调用相应的Eval()方法, 最后得到abs的对象表达: object.Object
```go
func Eval(node ast.Node, env *object.Environment) object.Object {
    //node重新赋值为实现了node的实体类型
    switch node := node.(type) {
    // Statements
    case *ast.Program:
        return evalProgram(node, env)

    case *ast.BlockStatement:
        return evalBlockStatement(node, env)
        
    //很多地方都是再次调用Eval
    //首先, *ast.ExpressionStatement实现了node的接口, 是node
    //其实体包括一个token和一个Expression
    //但这里直接忽略其本体实现, 直接使用了其Expression域
    //type ExpressionStatement struct {
    //    Token      token.Token // the first token of the expression
    //    Expression Expression
    //}
    case *ast.ExpressionStatement:
        //如上文交代的, Expression是node的扩展抽象. 很多基础expression都实现了这个抽象, 比如*ForInExpression或者*NumberLiteral
        //再次进Eval获取实体类型, 就会跑到其实体类型对应的case里面去
        return Eval(node.Expression, env)

    case *ast.ReturnStatement:
        val := Eval(node.ReturnValue, env)
        if isError(val) {
            return val
        }
        return &object.ReturnValue{Value: val}

    case *ast.AssignStatement:
        err := evalAssignment(node, env)

        if isError(err) {
            return err
        }

        return NULL
    // Expressions
    case *ast.NumberLiteral:
        return &object.Number{Token: node.Token, Value: node.Value}

    case *ast.NullLiteral:
        return NULL

    case *ast.CurrentArgsLiteral:
        return &object.Array{Token: node.Token, Elements: env.CurrentArgs, IsCurrentArgs: true}

    case *ast.StringLiteral:
        return &object.String{Token: node.Token, Value: util.InterpolateStringVars(node.Value, env)}

    case *ast.Boolean:
        return nativeBoolToBooleanObject(node.Value)

    case *ast.PrefixExpression:
        right := Eval(node.Right, env)
        if isError(right) {
            return right
        }
        return evalPrefixExpression(node.Token, node.Operator, right)

    case *ast.InfixExpression:
        return evalInfixExpression(node.Token, node.Operator, node.Left, node.Right, env)

    case *ast.CompoundAssignment:
        return evalCompoundAssignment(node, env)

    case *ast.IfExpression:
        return evalIfExpression(node, env)

    case *ast.WhileExpression:
        return evalWhileExpression(node, env)

    case *ast.ForExpression:
        return evalForExpression(node, env)

    case *ast.ForInExpression:
        return evalForInExpression(node, env)

    case *ast.Identifier:
        return evalIdentifier(node, env)

    case *ast.FunctionLiteral:
        params := node.Parameters
        body := node.Body
        name := node.Name
        fn := &object.Function{Token: node.Token, Parameters: params, Env: env, Body: body, Name: name, Node: node}

        if name != "" {
            env.Set(name, fn)
        }

        return fn

    case *ast.Decorator:
        return evalDecorator(node, env)

    case *ast.CallExpression:
        //对应函数调用
        function := Eval(node.Function, env)
        if isError(function) {
            return function
        }

        args := evalExpressions(node.Arguments, env)

        // Did we pass arguments as ...?
        // If so, replace arguments with the
        // environment's CurrentArgs.
        // If other arguments were passed afterwards
        // (eg. func(..., x, y)) we also add those.
        if len(args) > 0 {
            firstArg, ok := args[0].(*object.Array)

            if ok && firstArg.IsCurrentArgs {
                newArgs := env.CurrentArgs
                args = append(newArgs, args[1:]...)
            }
        }

        if len(args) == 1 && isError(args[0]) {
            return args[0]
        }

        return applyFunction(node.Token, function, env, args)

    case *ast.MethodExpression:
        o := Eval(node.Object, env)
        if isError(o) {
            return o
        }

        args := evalExpressions(node.Arguments, env)
        if len(args) == 1 && isError(args[0]) {
            return args[0]
        }

        return applyMethod(node.Token, o, node, env, args)

    case *ast.PropertyExpression:
        return evalPropertyExpression(node, env)

    case *ast.ArrayLiteral:
        elements := evalExpressions(node.Elements, env)
        if len(elements) == 1 && isError(elements[0]) {
            return elements[0]
        }
        return &object.Array{Token: node.Token, Elements: elements}

    case *ast.IndexExpression:
        return evalIndexExpression(node, env)

    case *ast.HashLiteral:
        return evalHashLiteral(node, env)

    case *ast.CommandExpression:
        //这个函数也很经典!
        //执行shell命令. 系统中必须有bash, 所有命令都是bash -c运行的
        //对输入的string类型的命令, 利用正则技术, 找出里面的所有变量, 在env里查找到变量值后, 替换string中的$VAR和${VAR}; 注意这里是把abs的变量展开为字符替换到cmd string中, 由bash去解析
        //如果命令string cmd以 &结尾, 说明是要后台执行; 注意这里, abs会自己处理后台的操作, 如果命令后台执行, 需要用字符串的Wait()方法来等待.
        //利用标准库的exec包来执行cmd, 把stdout stderror都定向到内部bytes.Buffer中
        return evalCommandExpression(node.Token, node.Value, env)

    // break and continue are treated just like errors: they will stop
    // the execution of the current code. Within FOR blocks, though, they
    // are caught and handled accordingly (see evalForExpression).
    case *ast.BreakStatement:
        return newBreakError(node.Token, "break called outside of a loop")
    // break and continue are treated just like errors: they will stop
    // the execution of the current code. Within FOR blocks, though, they
    // are caught and handled accordingly (see evalForExpression).
    case *ast.ContinueStatement:
        return newContinueError(node.Token, "continue called outside of a loop")

    }

    return NULL
}
```
以上可以看到, Eval()函数根据node的类型断言, 调用相应的实体类型的实现函数; 而在实体类型中, 很多也是包含了node类型的struct, 再次调用Eval()来获取其子实体类型

```go
type typeA struct {
    others Other
    //fieldX 是个interface
    fieldX node
}

type typeB struct {
    others Other
    //fieldY 是个interface
    fieldY node
}

func Eval(node ast.Node, env *object.Environment) object.Object {
    switch node := node.(type) {
    // typeA是struct类型, 实体类型
    case typeA:
        //fieldX是个node类型的interface
        Eval(node.fieldX, env)
    case typeB:
        Eval(node.fieldY, env)
    }
}
```
**所以, 一次递归的Eval()调用, 就对应了实体strcut{}中嵌套的一个node interface变量. 在设计node的时候, 实体类型里嵌套包括node的interface, 再递归调用Eval(), 这是个很精妙的框架设计**

## ast
ast对语法树的抽象是:
```go
//基本的node只有两个方法, 都返回string
// The base Node interface
type Node interface {
    TokenLiteral() string
    String() string
}

// All statement nodes implement this
type Statement interface {
    Node
    statementNode()
}

// All expression nodes implement this
type Expression interface {
    Node
    expressionNode()
}

//program就是statement的集合
// Represents the whole program
// as a bunch of statements
type Program struct {
    Statements []Statement
}
```
比如for的表达是
```go
type ForExpression struct {
    Token      token.Token     // The 'for' token
    Identifier string          // "x"
    Starter    Statement       // x = 0
    Closer     Statement       // x++
    Condition  Expression      // x < 1
    Block      *BlockStatement // The block executed inside the for loop
}

func (fe *ForExpression) expressionNode()      {}
func (fe *ForExpression) TokenLiteral() string { return fe.Token.Literal }
//看起来一个for表达的String()方法返回整个for的语句块.
func (fe *ForExpression) String() string {
    var out bytes.Buffer

    out.WriteString("for ")

    out.WriteString(fe.Starter.String())
    out.WriteString(";")
    out.WriteString(fe.Condition.String())
    out.WriteString(";")
    out.WriteString(fe.Closer.String())
    out.WriteString(";")
    out.WriteString(fe.Block.String())

    return out.String()
}
```

## 总结
相对yaegi, abs更倾向于使用interface来抽象, 看起来更简洁. 比如对node的抽象, abs就是个interface, 只规定了两个实现函数; 而yaegi则使用了传统的struct类型的node, 缺点是必须在这个struct里面, 定义一组"大而全"的字段, 以适应所有情况. 在某些情况下, 某些字段是无意义的.

# abs
看了yaegi和govaluate的实现, 先简单对比一下:
* yaegi: 基于go标准库的词法解析树, 最底层利用reflect.Value等反射方法来执行
    * 只依赖标准库, 但受限于此, 只能解释go的语法, 支持所有go语言的特性.
    * 完成度比较高, 能解释执行go代码.
    * 利用反射和go的runtime交互, 底层执行靠runtime的反射来实现. 比如调用fmt.Println实际上是调用的是已经编译好的runtime内部的符号.
* govaluate: 自造词法解析, 构建执行流, 底层直接调用基本函数.
    * 所谓的基本函数就是`left.(float64) * right.(float64)`的运算
    * 只支持简单的"表达式"计算, 侧重于得到表达式的值.
    
abs是个自定义语法的完整解释器实现. 下面我们来一探究竟.

## abs -- go实现的类shell
看起来非常不错!!!!!  
库地址: https://github.com/abs-lang/abs  
主页: https://www.abs-lang.org

[作者简介, 大神](https://odino.org/about/)
解释器基于[Thorsten Ball](https://twitter.com/thorstenball)的[Writing An Interpreter In Go](https://interpreterbook.com/), 另外还有一本[Writing A Compiler In Go](https://compilerbook.com/)

还有个类似的用go写的解释器:[Magpie Programming Language](https://github.com/haifenghuang/magpie) 是国人写的

[reddit讨论](https://www.reddit.com/r/ProgrammingLanguages/comments/ari17o/the_abs_programming_language/)

特点如下:
* 解释执行
* 兼备shell的灵活性和通用语言的准确性

first look
```shell
# Simple program that fetches your IP and sums it up
res = `curl -s 'https://api.ipify.org?format=json'`

if !res.ok {
  echo("An error occurred: %s", res)
  exit(1)
}

ip = res.json().ip
total = ip.split(".").map(int).sum()
if total > 100 {
    echo("The sum of [$ip] is a large number, $total.")
}
```

## 字符串
字符串是用引号括起来的:
```shell
"hello world"
'hello world'
```
用转义可以转义引号:
```shell
"I said: \"hello world\""
```
字符串支持切片操作, 也支持python一样的-index; 也支持+操作
```shell
"hello world"[1] # e
"string"[-2] # "n"
"string"[0:3] // "str"
"string"[0:3] // "str"
"string"[1:] // "tring"
"string"[0:-1] // "strin"
"hello" + " " + "world" # "hello world"
```
支持in操作
```shell
"str" in "string"   # true
"xyz" in "string"   # false
```

在字符串中引用变量要加`$`前缀, 比如
```shell
file = "/etc/hosts"
x = "File name is: $file"
echo(x) # "File name is: /etc/hosts"
```
或者在命令里面引用变量也要加`$`
```shell
var = "/etc"
out = `ls -la $var`
echo(out)
```

## 内置通用array结构
array是个万能容器
```shell
# array
[1, 2, "hello", [1, f(x){ x + 1 }]]
# array提供很多fancy的内置函数
[0, 1, 2].every(f(x){x == 0}) # false
[1, 2, 3].diff([3, 1]) # [2]
[[1, 2], 3, [4]].flatten() # [1, 2, 3, 4]
[[[1, 2], [[[[3]]]], [4]]].flatten_deep() # [1, 2, 3, 4]
[1, 2, 3].join("_") # "1_2_3"
[1, 2, 3].intersect([3, 1]) # [1, 3]
(1..2).keys() # [0, 1]
[1, 2].len() # 2
#每个元素调用函数
[0, 1, 2].map(f(x){x+1}) # [1, 2, 3]
[0, 5, -10, 100].max() # 100
f odd(n) {
  return !!(n % 2)
}
[0, 1, 2, 3, 4, 5].partition(odd) # [[0, 2, 4], [1, 3, 5]]

# pop
a = [1, 2, 3]
a.pop() # 3
a # [1, 2]

# shift
a = [1, 2, 3]
a.shift() # 1
a # [2, 3]

["b", "a", "c"].sort() # ["a", "b", "c"]
[1, 1, 1, 2].unique() # [1, 2]

```

## 内置通用hash结构, 不同于go的map, 这里的key只能是string

```
h = {"a": 1, "b": 2, "c": 3}
h # {a: 1, b: 2, c: 3}

# index assignment
h["a"] = 99
h # {a: 99, b: 2, c: 3}

# property assignment
h.a # 99
h.a = 88
h # {a: 88, b: 2, c: 3}

# compound operator assignment to property
h.a += 1
h.a # 89
h # {a: 88, b: 2, c: 3}

# create new keys via index or property
h["x"] = 10
h.y = 20
h # {a: 88, b: 2, c: 3, x: 10, y: 20}

h = {"a": 1, "b": 2, "c": 3}
h   # {a: 1, b: 2, c: 3}

# extending a hash by += compound operator
h += {"c": 33, "d": 4, "e": 5}
h   # {a: 1, b: 2, c: 33, d: 4, e: 5}

# 也有很多fancy的内置函数
h = {"a": 1, "b": 2, "c": 3}
h.items()   # [[a, 1], [b, 2], [c, 3]]
items(h)    # [[a, 1], [b, 2], [c, 3]]

h = {"a": 1, "b": 2, "c": 3}
h.keys() # [a, b, c]
keys(h) # [a, b, c]

h = {"a": 1, "b": 2, "c": {"x": 10, "y":20}}
h.pop("a")  # {a: 1}
h   # {b: 2, c: {x: 10, y: 20}}

h = {"a": 1, "b": 2, "c": 3}
h.values()  # [1, 2, 3]
values(h)   # [1, 2, 3]

hash = {"greeter": f(name) { return "Hello $name!" }}
hash.greeter("Sally") # "Hello Sally!"
```

## 函数

```shell
#有名字
f greet(name) {
    echo("Hello $name!")
}

greet(`whoami`) # "Hello root!"

#没名字
f(x, y) {
    x + y
}
#带不带return都行
f(x, y) {
    return x + y
}

#匿名函数
[1, 2, 3].map(f(x){ x + 1}) # [2, 3, 4]

#一等公民
func = f(x){ x + 1}
[1, 2, 3].map(func) # [2, 3, 4]

#默认参数
f greet(name, greeting = "hello") {
    echo("$greeting $name!")
}
greet("user") # hello user!
greet("user", "hola") # hola user!

#使用...特殊变量做变长参数
f sum_numbers() {
    s = 0
    for x in ... {
        s += x
    }

    return s
}

sum_numbers(1) # 1
sum_numbers(1, 2, 3) # 6

f echo_wrapper() {
    echo(..., "root")
}

echo_wrapper("hello %s %s", "sir") # "hello sir root"
```

## 内置函数

```shell
type(1) # NUMBER
arg(n)
args()
["abs", "--flag1", "--flag2", "arg1", "arg2"]
cd(path)
#echo直接支持格式化打印
# echo实际上底层调用的是fmt.Fprintf(env.Writer, args[0].Inspect(), arguments...)
# 见evaluator/functions.go
echo("hello %s", "world")
env("PATH") # "/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
#执行文本
eval("1 + 1") # 2
eval('object = {"x": 10}; object.x') # 10
exit(code [, message])
#参数
abs --test --test2 2 --test3=3 --test4 -test5
flag("test2")
2
pwd()
#随机数
rand(10) # 7
#支持引入外部脚本
mod = require("module.abs")
echo(mod.adder(1, 2)) # 3
#展开其他脚本在本脚本
source(path_to_file.abs)
#睡眠
sleep(1000) # sleeps for 1 second
#读取输入
stdin()
#从epoch开始的时间戳
unix_ms()
```

## 装饰器
竟然支持装饰器!
```shell
f uppercase(fn) {
    return f() {
        return fn(...).upper()
    }
}

@uppercase
f stringer(x) {
    return x.str()
}

stringer({}) # "{}"
stringer(12) # "12"
stringer("hello") # "HELLO"
```

如果函数执行太久就打印
```shell
f log_if_slow(treshold_ms) {
    return f(original_fn) {
        return f() {
            start = `date +%s%3N`.int()
            res = original_fn(...)
            end = `date +%s%3N`.int()

            if end - start > treshold_ms {
                echo("mmm, we were pretty slow...")
            }

            return res
        }
    }
}

@log_if_slow(500)
f return_random_number_after_sleeping(seconds) {
    `sleep $seconds`
    return rand(1000)
}
```

## 自带的标准库
abs自带标准库, 也是用require来引用
```shell
mod = require('@module')  # Loads "module" from the standard library
mod = require('./module') # Loads "module" from the current directory
mod = require('module')   # Loads "module" that was installed through the ABS package manager
```
目前的标准库只有cli runtime 和util

### cli
cli库本身只有100行!
```shell
cli = require('@cli')
#注册一个cmd
@cli.cmd("date", "prints the current date", {format: ''})
f date(args, flags) {
    format = flags.format
    return `date ${format}`
}
#运行这个cli
cli.run()
#交互式运行cli
cli.repl()
```
#### cli举例
```
#!/usr/bin/env abs
cli = require('@cli')

@cli.cmd("ip", "finds our IP address", {})
f ip_address(arguments, flags) {
    return `curl icanhazip.com`
}

@cli.cmd("date", "Is it Friday already?", {"format": ""})
f date(arguments, flags) {
    format = flags.format
    return `date ${format}`
}

cli.run()
```
把上面脚本保存为`./cli`, 调用这个脚本
```shell
$ ./cli 
Available commands:

  * date - Is it Friday already?
  * help - print this help message
  * ip - finds our IP address

$ ./cli help
Available commands:

  * date - Is it Friday already?
  * help - print this help message
  * ip - finds our IP address

$ ./cli ip  
87.201.252.69

$ ./cli date
Sat Apr  4 18:06:35 +04 2020

$ ./cli date --format +%s
1586009212
```

#### repl cli 举例
```shell
#!/usr/bin/env abs
cli = require('@cli')

res = {"count": 0}

@cli.cmd("count", "prints a counter", {})
f counter(arguments, flags) {
    echo(res.count)
}

@cli.cmd("incr", "Increment our counter", {})
f incr(arguments, flags) {
    res.count += 1
    return "ok"
}

@cli.cmd("incr_by", "Increment our counter", {})
f incr_by(arguments, flags) {
    echo("Increment by how much?")
    n = stdin().number()
    res.count += n
    return "ok"
}

cli.repl()
```
使用
```shell
$ ./cli 
help
Available commands:

  * count - prints a counter
  * help - print this help message
  * incr - Increment our counter
  * incr_by - Increment our counter
count
0
incr
ok
incr
ok
count
2
incr_by
Increment by how much?
-10
ok
count
-8
```

### 函数cache
比如一个函数要算个东西, 很久. 第一次执行的时候算, 后面如果入参都一样, 就直接用cache里面保存的. 这里的cache是指`util.memoize`这个装饰器
```shell
util = require('@util')
@util.memoize(60)
f expensive_task(x, y, z) {
    # do something very expensive here...
}
```
