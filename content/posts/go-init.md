---
title: "Golang，从 init 函数的顺序问题到 Go 编译器"
draft: false
tags: ["Golang", "编译器", "init function"]
---

在同一个 go 文件里，初始相关的执行顺序是 const -> var -> init()。显然，如果同文件里有多个 init()，那么将按照声明顺序来执行。如果是不同 package 里的 init()，那么将按照 import 的顺序来执行。下面将考虑更特殊的一些情况。

有如下场景：

```text
main.go
a/b.go
a/c.go
```

```go
// main.go
package main
import "go-init/a"
func main() {
    a.A()
}
```

```go
// a/b.go
package a

func init() {
    println("b init")
}

func A() int {
    println("A")
    return 0
}
```

```go
// a/c.go
package a

func init() {
    println("c init")
}
```

执行 `go run main.go`，输出：

```text
b init
c init
A
```

然后将 `a/b.go` 改名为 `a/d.go`，再次执行 `go run main.go`，输出：

```text
c init
b init
A
```

可以看到有 [现象1]：`a/b.go` 和 `a/c.go` 的 init() 函数的执行顺序是按照文件名的字母顺序来的，将 `a/b.go` 改名后，其文件名顺序排在了 `a/c.go` 之后，最终 init() 执行也排在了之后。

> 如果 package 内的文件存在 "引用关系" 呢？

我们在 `a/c.go` 中加入 `var C = A()`，由于 A() 是在 `a/d.go` 中定义的，所以会先处理 `a/d.go` 吗？

```text
A
c init
b init
A
```

从结果可见[现象2]：就算 var 先进行初始化，使用了定义在 `a/d.go` 中的 A()，但是 init() 函数的执行顺序仍然是按照文件名的字母顺序来的。

类似的，还有[现象3]：package 内文件中包含的 import 的处理顺序也是按照文件名的字母顺序来的，跟这里的 init() 函数的执行顺序是类似的，也就是说： 相同 package 内，a.go 内的 import “似乎”会先于 b.go 内的 import 执行。

实际上，导致这些结果的原因均来自于编译器对文件的处理，从源码中能够找到这三个现象的根源。

Golang 编译器相关源码位于 [`go/src/cmd/compile/`](https://github.com/golang/go/tree/d8117459c513e048eb72f11988d5416110dff359/src/cmd/compile)。

> 本文中引用的源码均标注了链接，版本为 `Go 1.21 release`。

Go 编译处理的单位是 Package，生成的结果是对象文件。在一次编译过程开始时会读取 Package 中所有文件内容进行词法和语法分析。我们很容易就能找到编译器的入口：

```go
// https://github.com/golang/go/blob/d8117459c513e048eb72f11988d5416110dff359/src/cmd/compile/main.go#L45
func main() {
    // disable timestamps for reproducible output
    log.SetFlags(0)
    log.SetPrefix("compile: ")

    buildcfg.Check()
    archInit, ok := archInits[buildcfg.GOARCH]
    if !ok {
        fmt.Fprintf(os.Stderr, "compile: unknown architecture %q\n", buildcfg.GOARCH)
        os.Exit(2)
    }

    gc.Main(archInit)
    base.Exit(0)
}
```

此处发生在 [noder.LoadPackage(flag.Args())](https://github.com/golang/go/blob/d8117459c513e048eb72f11988d5416110dff359/src/cmd/compile/internal/noder/noder.go#L27)，每个文件的词法和语法分析是并行处理的，[]*noder 的顺序也就是所给出的文件列表的顺序，noder 对应单一的文件，是这个文件中代码的 AST，可见在经过词法和语法分析时，仍是对单个文件进行的处理，还没有决定 init 的顺序，更没有进行 import 引入其他 Package。

接下来 []*noder 被传入 check2() 用于类型检查。

```go
// check2 type checks a Go package using types2, and then generates IR
// using the results.
func check2(noders []*noder) {
    m, pkg, info := checkFiles(noders)
    g := irgen{
        target: typecheck.Target,
        self:   pkg,
        info:   info,
        posMap: m,
        objs:   make(map[types2.Object]*ir.Name),
        typs:   make(map[types2.Type]*types.Type),
    }
    g.generate(noders)
}
```

注意 checkFiles() 返回了自身这个 Package 的信息 pkg，其中包含了所有 import 的 Package 的信息。后续 g.generate() 是在类型检查结束后生成 IR 树，g.generate() 中仍然依次处理 noder，在遇到 init 函数后会加入全局变量 typecheck.Target.Inits 列表，因此列表中 init 加入的顺序是先按照文件名顺序，再按照文件中的位置顺序。

IR 树生成过程中遇到函数是这样处理的：

```go
case pkgbits.ObjFunc:
    if sym.Name == "init" {
        sym = Renameinit()
    }
...
```

可见 init 函数是多么特殊，它会被重命名，这样就不会与其他 init 函数冲突了。Renameinit() 的实现如下：

```go
var renameinitgen int

func Renameinit() *types.Sym {
    s := typecheck.LookupNum("init.", renameinitgen)
    renameinitgen++
    return s
}
```

可见只是给了个编号，重命名成了一系列 init.0 init.1 init.2 等等的函数。

```go
// A Package describes a Go package.
type Package struct {
    path     string
    name     string
    scope    *Scope
    imports  []*Package
    complete bool
    fake     bool // scope lookup errors are silently dropped if package is fake (internal use only)
    cgo      bool // uses of this package will be rewritten into uses of declarations from _cgo_gotypes.go
}
```

也就是说终于在 checkFiles() 时，加载了其他的 Package。接下来看看被引用的 Package 是如何被加载的，源码见 `go/src/cmd/compile/internal/noder/irgen.go`。

```go
importer := gcimports{
    ctxt:     ctxt,
    packages: make(map[string]*types2.Package),
}
conf := types2.Config{
    ...
    Importer:               &importer,
    ...
}
...
pkg, err := conf.Check(base.Ctxt.Pkgpath, files, info)
```

其中 gcimports 提供了一个 Import 方法，用于加载其他 Package，将会在 conf.Check() 中被调用。这个方法实际上是将编译好的 Package 对象文件重新加载为 types2.Package 结构体，也就是编译成对象文件的逆操作，显然这要求被依赖的 Package 提前编译好。

```go
func (conf *Config) Check(path string, files []*syntax.File, info *Info) (*Package, error) {
    pkg := NewPackage(path, "")
    return pkg, NewChecker(conf, pkg, info).Files(files)
}
```

conf.Check() 简单新建了一个 Package 结构用于表示自身，然后新建了一个 Checker，并调用了 Checker 的 Files() 方法，可以断定 import 的处理是在 Checker 的 Files() 方法中进行的，我们继续深入。

```go
...
print("== initFiles ==")
check.initFiles(files)

print("== collectObjects ==")
check.collectObjects()

print("== packageObjects ==")
check.packageObjects()

print("== processDelayed ==")
check.processDelayed(0) // incl. all functions

print("== cleanup ==")
check.cleanup()

print("== initOrder ==")
check.initOrder()
...
```

initFiles() 用于检查文件开头的 package 语句所声明的名称是否符合要求，例如要跟当前 package 名一致，否则忽略这个文件（都经过词法语法分析了，白分析了，当然编译前就能检查出这些问题，一般不会进行到这里才发现）。

collectObjects() 在此处对 import 的 Package 进行了加载，并将其置于相应的 Scope 中。可以看到这里仍然是按照文件顺序在进行处理，通过 `check.impMap` 来记录已经 import 的 Package，避免重复加载 Package；同时用 `var pkgImports = make(map[*Package]bool)` 来记录本 Package 所引用的所有 Package。

同时，还能从中看到一些特殊 import 的处理，例如 import . 和 import _ 以及别名。DotImport 会将 imported package 中的导出符号全部遍历导入到当前的 FileScope 中，而一般情况下是将 imported package 整个加入到当前的 FileScope 中，这样会有额外的层次结构。

> 注意这里提到了 FileScope，我们知道在 Go 的同一个 Package 下，许多声明是不存在 FileScope 的，例如全局变量在一个文件中声明，另一个文件中可以直接使用；同名也会发生冲突，因为这些都在同一个 PackageScope 下。但是对于 import 操作来说，每个文件都有自己需要 import 的内容，因此需要一个 FileScope 来记录区分这些信息。

Scope 结构组织好后，还需要检查 FileScope 跟 PackageScope 之间的冲突问题，这主要是 DotImport 导致的。

```go
 // verify that objects in package and file scopes have different names
for _, scope := range fileScopes {
    for name, obj := range scope.elems {
        if alt := pkg.scope.Lookup(name); alt != nil {
            ...
```

initOrder() 是对一些有依赖关系的全局声明进行排序，并未涉及 init() 的处理，例如：

```go
var (
    // a depends on b and c, c depends on f
    a = b + c
    b = 1
    c = f()

    // circular dependency
    d = e
    e = d
)
```

在 Go 中，能够被用于初始化表达式的对象被称为 Dependency 对象，有 Const, Var, Func 这三类。先构建对象依赖关系的有向图（Directed Graph），再以每个节点的依赖数目为权重构建最小堆（MinHeap）并以此堆作为最小优先级队列（PriorityQueue），因此队列头部的对象总是依赖其它对象最少的，所以该队列的遍历顺序就是初始化的顺序，是很常规的处理思路。要注意常量的初始化比较简单，在类型检查时就已经确定，在这里仍然加入是为了检测循环依赖。

于是乎，类型检查得到的 IR 树便是包含了完整的 import 信息的，接下来便会进行 init() 相关的处理。要注意的是 Package 里面特地用 []*Package 的形式保存了所有 import 的 Package。

接下来终于来到了 pkginit 包的内容。

```go
// Parse and typecheck input.
noder.LoadPackage(flag.Args())
...
// Create "init" function for package-scope variable initialization
// statements, if any.
//
// Note: This needs to happen early, before any optimizations. The
// Go spec defines a precise order than initialization should be
// carried out in, and even mundane optimizations like dead code
// removal can skew the results (e.g., #43444).
pkginit.MakeInit()
```

从注释也可以知道，在词法分析、语法分析以及类型检查和构造 IR 树的过程中，均未涉及代码优化。以下是 MakeInit() 的内容，关键部分使用中文进行了更详细的注释，可以对照相关方法的源码进行阅读。

```go
// TODO(mdempsky): Move into noder, so that the types2-based frontends
// can use Info.InitOrder instead.
func MakeInit() {
    // Init 相关的处理只涉及全局声明（Package Level），依赖关系作为有向边来构建有向图，然后进行拓扑排序。
    nf := initOrder(typecheck.Target.Decls)
    if len(nf) == 0 {
        return
    }

    // Make a function that contains all the initialization statements.
    base.Pos = nf[0].Pos() // prolog/epilog gets line number of first init stmt
    // 查找 init 符号，如果 Package 的全局符号中没有则创建；不用担心 init 符号已经被用户的 init 函数使用，因为 IR 树在生成过程中遇到 init 会重命名为 init.0 init.1 这样的格式，前面提到 g.generate() 的时候也有说明。
    initializers := typecheck.Lookup("init")
    /* 用 init 符号声明一个新的函数，用于存放所有的初始化工作。具体实现是：此处在 IR 树中对应位置建立了新的 ONAME Node（ONAME 表示 var/func name），类型指定为 PFUNC，同时也将符号表中的 init 更新为 symFunc，表明这个符号是函数名；然后新建一个函数节点，将 ONAME Node 指向函数节点，最后将函数节点返回。
    */
    fn := typecheck.DeclFunc(initializers, nil, nil, nil)
    // 类型检查过程中生成了一个 InitTodoFunc，其作为全局初始化语句的临时上下文环境。现在将临时环境 InitTodoFunc 的内容转移到 fn。
    for _, dcl := range typecheck.InitTodoFunc.Dcl {
        dcl.Curfn = fn
    }
    fn.Dcl = append(fn.Dcl, typecheck.InitTodoFunc.Dcl...)
    typecheck.InitTodoFunc.Dcl = nil

    // Suppress useless "can inline" diagnostics.
    // Init functions are only called dynamically.
    fn.SetInlinabilityChecked(true)

    // 配置函数体。
    fn.Body = nf
    typecheck.FinishFuncBody()

    // 确定 fn 为函数节点
    typecheck.Func(fn)
    // 在 fn 的内部上下文环境下检查函数体
    ir.WithFunc(fn, func() {
        typecheck.Stmts(nf)
    })
    // 把函数加入到 Package 的全局声明列表。
    typecheck.Target.Decls = append(typecheck.Target.Decls, fn)

    // Prepend to Inits, so it runs first, before any user-declared init
    // functions.
    typecheck.Target.Inits = append([]*ir.Func{fn}, typecheck.Target.Inits...)

    if typecheck.InitTodoFunc.Dcl != nil {
        // We only generate temps using InitTodoFunc if there
        // are package-scope initialization statements, so
        // something's weird if we get here.
        base.Fatalf("InitTodoFunc still has declarations")
    }
    typecheck.InitTodoFunc = nil
}
```

旧版本中 MakeInit() 的工作是在 pkginit.Task() 中实现的，现在被抽取了出来，原因有以下几点。

1. 首先，MakeInit() 负责初始化函数的创建并插入 typecheck.Target.Inits，pkginit.Task() 得到了简化，毕竟这个初始化函数和其他用户定义的 init 实际上没有本质区别。

2. 其次，敏锐的同学可能发现了，类型检查的过程中，已经进行了一次 initOrder()，但只检查了循环依赖的问题；这次又 initOrder() 显得有些冗余。因此将这一部分从 Task() 中拆分出来，希望以后能够放到类型检查的过程中，避免重复的排序操作。当前抽离成了单独的函数但是还未并入类型检查，处于中间状态，可见不久后将会并入类型检查，注释中的 TODO 就是在说这个问题。

3. 最后，初始化函数如果在 Task() 中创建，则无法参与到类型检查结束到 Task() 开始这之间的优化过程，主要包括无效代码清理和内联优化。因此将其提前到类型检查结束后创建，这样就可以参与到优化过程中了。

最后，终于来到了 init 处理的终点， pkginit.Task()。

```go
// Task makes and returns an initialization record for the package.
// See runtime/proc.go:initTask for its layout.
// The 3 tasks for initialization are:
//  1. Initialize all of the packages the current package depends on.
//  2. Initialize all the variables that have initializers.
//  3. Run any init functions.
func Task() *ir.Name {
    ...
    // Find imported packages with init tasks.
    // 这里可以看出 Package 最终的初始化任务被合并在了 .inittask 这个结构体中，因此对于引用的包才能这样进行查找，此处还检查了 .inittask 结构体是否合法。最终加入到 deps 数组。
    for _, pkg := range typecheck.Target.Imports {
        n := typecheck.Resolve(ir.NewIdent(base.Pos, pkg.Lookup(".inittask")))
        if n.Op() == ir.ONONAME {
            continue
        }
        if n.Op() != ir.ONAME || n.(*ir.Name).Class != ir.PEXTERN {
            base.Fatalf("bad inittask: %v", n)
        }
        deps = append(deps, n.(*ir.Name).Linksym())
    }
    ...
    // 如果开启了 Address Sanitizer，那么需要在创建一个 init 函数加入 typecheck.Target.Inits，用于初始化 ASan 相关的全局变量。
    if base.Flag.ASan {
        ...
        // 可见这个 init 将会在最后执行
        typecheck.Target.Inits = append(typecheck.Target.Inits, fnInit)
    }
    ...
    // Record user init functions.
    for _, fn := range typecheck.Target.Inits {
        // 只有处理 Package 全局变量的才叫 init，其它的都被重命名为了 init.0、init.1 等。
        if fn.Sym().Name == "init" {
            // Synthetic init function for initialization of package-scope
            // variables. We can use staticinit to optimize away static
            // assignments.
            s := staticinit.Schedule{
                Plans: make(map[ir.Node]*staticinit.Plan),
                Temps: make(map[ir.Node]*ir.Name),
            }
            for _, n := range fn.Body {
                s.StaticInit(n)
            }
            fn.Body = s.Out
            ir.WithFunc(fn, func() {
                typecheck.Stmts(fn.Body)
            })

            if len(fn.Body) == 0 {
                fn.Body = []ir.Node{ir.NewBlockStmt(src.NoXPos, nil)}
            }
        }

        // Skip init functions with empty bodies.
        if len(fn.Body) == 1 {
            if stmt := fn.Body[0]; stmt.Op() == ir.OBLOCK && len(stmt.(*ir.BlockStmt).List) == 0 {
                continue
            }
        }
        fns = append(fns, fn.Nname.Linksym())
    }
```

最终 fns 数组保存了所有 init 函数；deps 数组保存了所有依赖的包的 .inittask 结构体。接下来合并构建自己 Package 的 .inittask。

```go
// Make an .inittask structure.
sym := typecheck.Lookup(".inittask")
task := typecheck.NewName(sym)
// 显然这个 .inittask 不是 uint8 类型的，只是为了占位，因此这里设置了一个 fake type。
task.SetType(types.Types[types.TUINT8]) // fake type
task.Class = ir.PEXTERN
sym.Def = task
lsym := task.Linksym()
ot := 0
// lsym.P = [0]
ot = objw.Uintptr(lsym, ot, 0) // state: not initialized yet
// lsym.P = [0, len(deps)]
ot = objw.Uintptr(lsym, ot, uint64(len(deps)))
// lsym.P = [0, len(deps), len(fns)]
ot = objw.Uintptr(lsym, ot, uint64(len(fns)))
// lsym.R = [newR(d)...]
for _, d := range deps {
    ot = objw.SymPtr(lsym, ot, d, 0)
}
// lsym.R = [newR(d)..., newR(f)...]
for _, f := range fns {
    ot = objw.SymPtr(lsym, ot, f, 0)
}
// An initTask has pointers, but none into the Go heap.
// It's not quite read only, the state field must be modifiable.
// 此处说明这个 .inittask 符号是全局的，决定了最后在 object 文件中的位置区域。
objw.Global(lsym, int32(ot), obj.NOPTR)
return task
```

在最后将其设置为导出的（Export），因为其符号名并非大写字母开头，但是要被其他包使用：

```go
// Build init task, if needed.
if initTask := pkginit.Task(); initTask != nil {
    typecheck.Export(initTask)
}
```

至此，Package 单元对于 init 的处理就结束了，最后 Package 被编译为带有 .inittask 表的 object 文件，这个表中包含了所有的 init.x 函数和依赖的包的 .inittask 结构体指针，要注意这里只知道符号之间的关系，其他包里 init 函数的具体实现是不知道的，需要在链接阶段处理。

链接时，在拥有了所有的 .inittask 包含的具体函数相关信息后，链接器会将其按照依赖关系进行排序，生成一个具体的 mainInittasks 列表供 runtime 使用。此处不再展开这一部分，有兴趣的同学可以自行阅读链接器 inittask 部分的源码：`src/cmd/link/internal/ld/inittask.go`，最终 SymbolName 为 `go:main.inittasks`。

最终链接生成可执行文件时，inittasks 会被设计为加载到 `src/runtime/proc.go` 的 runtime_inittasks 数组中，然后在`runtime.main` 函数中被使用：

```go
func main() {
    ...
    doInit(runtime_inittasks) // Must be before defer.
    ...
}
```

最后回看一开始发现的三个现象：

[现象1]：`a/b.go` 和 `a/c.go` 的 init() 函数的执行顺序是按照文件名的字母顺序来的，将 `a/b.go` 改名后，其文件名顺序排在了 `a/c.go` 之后，最终 init() 执行也排在了之后。

[现象2]：就算 var 先进行初始化，使用了定义在 `a/d.go` 中的 A()，但是 init() 函数的执行顺序仍然是按照文件名的字母顺序来的。

[现象3]：package 内文件中包含的 import 的处理顺序也是按照文件名的字母顺序来的，跟这里的 init() 函数的执行顺序是类似的，也就是说： `相同 package 内，a.go 内的 import 会先于 b.go 内的 import 执行`。

根源在于编译器在读取源文件时是按照文件系统文件名顺序读入，在处理时也是依文件次序处理的，也就是编译器遇到 init 和 import 的顺序都是由文件名顺序决定的。

1. 虽然有 initOrder() 的存在，但是它不会影响用户定义的 init() 的顺序，因此 [现象1] 和 [现象2] 是肯定的。

2. initOrder() 会处理 import 的依赖关系，因此 [现象3] 只会在 a.go 和 b.go 所 import 的包之间不存在依赖关系时才成立。如果有：

```go
// a1.go
import "b"

// a2.go
import "c"

// b.go
import "c"
```

那么不管 a1.go 和 a2.go 的文件名顺序如何，a2.go 内的 import 语句都会先于 a2.go 内的 import 初始化，因为 b 依赖 c。
