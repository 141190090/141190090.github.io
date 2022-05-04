# golang底层数据结构

## 数组
数组是由相同类型元素的集合组成的数据结构，计算机会为数组分配一块连续的内存来保存其中的元素，我们可以利用数组中元素的索引快速访问特定元素。数组作为一种基本的数据类型，我们通常会从两个维度描述数组，也就是数组中存储的元素类型和数组最大能存储的元素个数，在 Go 语言中我们往往会使用如下所示的方式来表示数组类型：
```
[10]int
[200]interface{}
```
Go 语言数组在初始化之后大小就无法改变，存储元素类型相同、但是大小不同的数组类型在 Go 语言看来也是完全不同的，只有两个条件都相同才是同一类型。

### 数组的初始化
```
// Array contains Type fields specific to array types.
type Array struct {
	Elem  *Type // element type
	Bound int64 // number of elements; <0 if unknown yet
}

// NewArray returns a new fixed-length array Type.
func NewArray(elem *Type, bound int64) *Type {
	if bound < 0 {
		base.Fatalf("NewArray: invalid bound %v", bound)
	}
	t := newType(TARRAY)
	t.extra = &Array{Elem: elem, Bound: bound}
	t.SetNotInHeap(elem.NotInHeap())
	if elem.HasTParam() {
		t.SetHasTParam(true)
	}
	if elem.HasShape() {
		t.SetHasShape(true)
	}
	return t
}
```
以golang1.18为例，数组是在编译期由https://github.com/golang/go/blob/go1.18/src/cmd/compile/internal/types/type.go
的NewArray 函数生成的，该类型包含两个字段，分别是元素类型 Elem 和数组的大小 Bound，这两个字段共同构成了数组类型，而当前数组是否应该在堆栈中初始化也在编译期就确定了。

Go 语言的数组有两种不同的创建方式，一种是显式的指定数组大小，另一种是使用 [...]T 声明数组，Go 语言会在编译期间通过源代码推导数组的大小：
```
arr1 := [3]int{1, 2, 3}
arr2 := [...]int{1, 2, 3}
```
上述两种声明方式在运行期间得到的结果是完全相同的，后一种声明方式在编译期间就会被转换成前一种，这也就是编译器对数组大小的推导，下面我们来介绍编译器的推导过程。

上限推导 #
两种不同的声明方式会导致编译器做出完全不同的处理，如果我们使用第一种方式 [10]T，那么变量的类型在编译进行到类型检查阶段就会被提取出来，随后使用 cmd/compile/internal/types.NewArray创建包含数组大小的 cmd/compile/internal/types.Array 结构体。

当我们使用 [...]T 的方式声明数组时，编译器会在的 https://github.com/golang/go/blob/4aa1efed4853ea067d665a952eee77c52faac774/src/cmd/compile/internal/typecheck/expr.go#L231 对该数组的大小进行推导：
```
// The result of tcCompLit MUST be assigned back to n, e.g.
// 	n.Left = tcCompLit(n.Left)
func tcCompLit(n *ir.CompLitExpr) (res ir.Node) {
	if base.EnableTrace && base.Flag.LowerT {
		defer tracePrint("tcCompLit", n)(&res)
	}

	lno := base.Pos
	defer func() {
		base.Pos = lno
	}()

	if n.Ntype == nil {
		base.ErrorfAt(n.Pos(), "missing type in composite literal")
		n.SetType(nil)
		return n
	}

	// Save original node (including n.Right)
	n.SetOrig(ir.Copy(n))

	ir.SetPos(n.Ntype)

	// Need to handle [...]T arrays specially.
	if array, ok := n.Ntype.(*ir.ArrayType); ok && array.Elem != nil && array.Len == nil {
		array.Elem = typecheckNtype(array.Elem)
		elemType := array.Elem.Type()
		if elemType == nil {
			n.SetType(nil)
			return n
		}
		length := typecheckarraylit(elemType, -1, n.List, "array literal")
		n.SetOp(ir.OARRAYLIT)
		n.SetType(types.NewArray(elemType, length))
		n.Ntype = nil
		return n
	}

	n.Ntype = typecheckNtype(n.Ntype)
	t := n.Ntype.Type()
	if t == nil {
		n.SetType(nil)
		return n
	}
	n.SetType(t)

	switch t.Kind() {
	default:
		base.Errorf("invalid composite literal type %v", t)
		n.SetType(nil)

	case types.TARRAY:
		typecheckarraylit(t.Elem(), t.NumElem(), n.List, "array literal")
		n.SetOp(ir.OARRAYLIT)
		n.Ntype = nil
    // ......
    }

	return n
}
```
typecheckarraylit通过遍历元素的方式来计算数组中元素的数量。

所以我们可以看出 `[...]T{1, 2, 3}` 和 `[3]T{1, 2, 3}` 在运行时是完全等价的，`[...]T` 这种初始化方式也只是 Go 语言为我们提供的一种语法糖，当我们不想计算数组中的元素个数时可以通过这种方法减少一些工作量。

语句转换 #
对于一个由字面量组成的数组，根据数组元素数量的不同，编译器会在负责初始化字面量的anylit函数中做两种不同的优化：

当元素数量小于或者等于 4 个时，会直接将数组中的元素放置在栈上；
当元素数量大于 4 个时，会将数组中的元素放置到静态区并在运行时取出赋值；
```
// https://github.com/golang/go/blob/go1.18/src/cmd/compile/internal/walk/complit.go
func anylit(n ir.Node, var_ ir.Node, init *ir.Nodes) {
	t := n.Type()
	switch n.Op() {
	default:
		base.Fatalf("anylit: not lit, op=%v node=%v", n.Op(), n)

    // ......
	case ir.OSTRUCTLIT, ir.OARRAYLIT:
		n := n.(*ir.CompLitExpr)
		if !t.IsStruct() && !t.IsArray() {
			base.Fatalf("anylit: not struct/array")
		}

		if isSimpleName(var_) && len(n.List) > 4 {
			// lay out static data
			vstat := readonlystaticname(t)

			ctxt := inInitFunction
			if n.Op() == ir.OARRAYLIT {
				ctxt = inNonInitFunction
			}
			fixedlit(ctxt, initKindStatic, n, vstat, init)

			// copy static to var
			appendWalkStmt(init, ir.NewAssignStmt(base.Pos, var_, vstat))

			// add expressions to automatic
			fixedlit(inInitFunction, initKindDynamic, n, var_, init)
			break
		}

		var components int64
		if n.Op() == ir.OARRAYLIT {
			components = t.NumElem()
		} else {
			components = int64(t.NumFields())
		}
		// initialization of an array or struct with unspecified components (missing fields or arrays)
		if isSimpleName(var_) || int64(len(n.List)) < components {
			appendWalkStmt(init, ir.NewAssignStmt(base.Pos, var_, nil))
		}

		fixedlit(inInitFunction, initKindLocalCode, n, var_, init)
    // ......
}

func fixedlit(ctxt initContext, kind initKind, n *ir.CompLitExpr, var_ ir.Node, init *ir.Nodes) {
    // ......
	for _, r := range n.List {
		a, value := splitnode(r)
		if a == ir.BlankNode && !staticinit.AnySideEffects(value) {
			// Discard.
			continue
		}

		switch value.Op() {
		case ir.OARRAYLIT, ir.OSTRUCTLIT:
			value := value.(*ir.CompLitExpr)
			fixedlit(ctxt, kind, value, a, init)
			continue
		}

		islit := ir.IsConstNode(value)
		if (kind == initKindStatic && !islit) || (kind == initKindDynamic && islit) {
			continue
		}

		// build list of assignments: var[index] = expr
		ir.SetPos(a)
		as := ir.NewAssignStmt(base.Pos, a, value)
		as = typecheck.Stmt(as).(*ir.AssignStmt)
		switch kind {
		case initKindStatic:
			genAsStatic(as)
		case initKindDynamic, initKindLocalCode:
			a = orderStmtInPlace(as, map[string][]*ir.Name{})
			a = walkStmt(a)
			init.Append(a)
		default:
			base.Fatalf("fixedlit: bad kind %d", kind)
		}

	}
}

```
当数组的元素小于或者等于四个时，kind为initKindLocalCode，会调用负责在函数编译之前将 [3]{1, 2, 3} 转换成更加原始的语句，拆分成一个声明变量的表达式和几个赋值表达式，这些表达式会完成对数组的初始化：

var arr [3]int
arr[0] = 1
arr[1] = 2
arr[2] = 3
但是如果当前数组的元素大于四个，cmd/compile/internal/gc.anylit 会先获取一个唯一的 staticname，然后调用 cmd/compile/internal/gc.fixedlit 函数在静态存储区初始化数组中的元素并将临时变量赋值给数组。
假设代码需要初始化 [5]int{1, 2, 3, 4, 5}，那么我们可以将上述过程理解成以下的伪代码：

var arr [5]int
statictmp_0[0] = 1
statictmp_0[1] = 2
statictmp_0[2] = 3
statictmp_0[3] = 4
statictmp_0[4] = 5
arr = statictmp_0
总结起来，在不考虑逃逸分析的情况下，如果数组中元素的个数小于或者等于 4 个，那么所有的变量会直接在栈上初始化，如果数组元素大于 4 个，变量就会在静态存储区初始化然后拷贝到栈上，这些转换后的代码才会继续进入中间代码生成和机器码生成两个阶段，最后生成可以执行的二进制文件。

### 数组的访问和赋值
无论是在栈上还是静态存储区，数组在内存中都是一连串的内存空间，我们通过指向数组开头的指针、元素的数量以及元素类型占的空间大小表示数组。如果我们不知道数组中元素的数量，访问时可能发生越界；而如果不知道数组中元素类型的大小，就没有办法知道应该一次取出多少字节的数据，无论丢失了哪个信息，我们都无法知道这片连续的内存空间到底存储了什么数据。
数组访问越界是非常严重的错误，Go 语言中可以在编译期间的静态类型检查判断数组越界，cmd/compile/internal/gc.typecheck1 会验证访问数组的索引：

func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	case OINDEX:
		ok |= ctxExpr
		l := n.Left  // array
		r := n.Right // index
		switch n.Left.Type.Etype {
		case TSTRING, TARRAY, TSLICE:
			...
			if n.Right.Type != nil && !n.Right.Type.IsInteger() {
				yyerror("non-integer array index %v", n.Right)
				break
			}
			if !n.Bounded() && Isconst(n.Right, CTINT) {
				x := n.Right.Int64()
				if x < 0 {
					yyerror("invalid array index %v (index must be non-negative)", n.Right)
				} else if n.Left.Type.IsArray() && x >= n.Left.Type.NumElem() {
					yyerror("invalid array index %v (out of bounds for %d-element array)", n.Right, n.Left.Type.NumElem())
				}
			}
		}
	...
	}
}
Go
访问数组的索引是非整数时，报错 “non-integer array index %v”；
访问数组的索引是负数时，报错 “invalid array index %v (index must be non-negative)"；
访问数组的索引越界时，报错 “invalid array index %v (out of bounds for %d-element array)"；
数组和字符串的一些简单越界错误都会在编译期间发现，例如：直接使用整数或者常量访问数组；但是如果使用变量去访问数组或者字符串时，编译器就无法提前发现错误，我们需要 Go 语言运行时阻止不合法的访问：

arr[4]: invalid array index 4 (out of bounds for 3-element array)
arr[i]: panic: runtime error: index out of range [4] with length 3
Go
Go 语言运行时在发现数组、切片和字符串的越界操作会由运行时的 runtime.panicIndex 和 runtime.goPanicIndex 触发程序的运行时错误并导致崩溃退出：

TEXT runtime·panicIndex(SB),NOSPLIT,$0-8
	MOVL	AX, x+0(FP)
	MOVL	CX, y+4(FP)
	JMP	runtime·goPanicIndex(SB)

func goPanicIndex(x int, y int) {
	panicCheck1(getcallerpc(), "index out of range")
	panic(boundsError{x: int64(x), signed: true, y: y, code: boundsIndex})
}
Go
当数组的访问操作 OINDEX 成功通过编译器的检查后，会被转换成几个 SSA 指令，假设我们有如下所示的 Go 语言代码，通过如下的方式进行编译会得到 ssa.html 文件：

package check

func outOfRange() int {
	arr := [3]int{1, 2, 3}
	i := 4
	elem := arr[i]
	return elem
}

$ GOSSAFUNC=outOfRange go build array.go
dumped SSA to ./ssa.html
Go
start 阶段生成的 SSA 代码就是优化之前的第一版中间代码，下面展示的部分是 elem := arr[i] 对应的中间代码，在这段中间代码中我们发现 Go 语言为数组的访问操作生成了判断数组上限的指令 IsInBounds 以及当条件不满足时触发程序崩溃的 PanicBounds 指令：

b1:
    ...
    v22 (6) = LocalAddr <*[3]int> {arr} v2 v20
    v23 (6) = IsInBounds <bool> v21 v11
If v23 → b2 b3 (likely) (6)

b2: ← b1-
    v26 (6) = PtrIndex <*int> v22 v21
    v27 (6) = Copy <mem> v20
    v28 (6) = Load <int> v26 v27 (elem[int])
    ...
Ret v30 (+7)

b3: ← b1-
    v24 (6) = Copy <mem> v20
    v25 (6) = PanicBounds <mem> [0] v21 v11 v24
Exit v25 (6)
Go
编译器会将 PanicBounds 指令转换成上面提到的 runtime.panicIndex 函数，当数组下标没有越界时，编译器会先获取数组的内存地址和访问的下标、利用 PtrIndex 计算出目标元素的地址，最后使用 Load 操作将指针中的元素加载到内存中。

当然只有当编译器无法对数组下标是否越界无法做出判断时才会加入 PanicBounds 指令交给运行时进行判断，在使用字面量整数访问数组下标时会生成非常简单的中间代码，当我们将上述代码中的 arr[i] 改成 arr[2] 时，就会得到如下所示的代码：

b1:
    ...
    v21 (5) = LocalAddr <*[3]int> {arr} v2 v20
    v22 (5) = PtrIndex <*int> v21 v14
    v23 (5) = Load <int> v22 v20 (elem[int])
    ...
Go
Go 语言对于数组的访问还是有着比较多的检查的，它不仅会在编译期间提前发现一些简单的越界错误并插入用于检测数组上限的函数调用，还会在运行期间通过插入的函数保证不会发生越界。

数组的赋值和更新操作 a[i] = 2 也会生成 SSA 生成期间计算出数组当前元素的内存地址，然后修改当前内存地址的内容，这些赋值语句会被转换成如下所示的 SSA 代码：

b1:
    ...
    v21 (5) = LocalAddr <*[3]int> {arr} v2 v19
    v22 (5) = PtrIndex <*int> v21 v13
    v23 (5) = Store <mem> {int} v22 v20 v19
    ...
Go
赋值的过程中会先确定目标数组的地址，再通过 PtrIndex 获取目标元素的地址，最后使用 Store 指令将数据存入地址中，从上面的这些 SSA 代码中我们可以看出 上述数组寻址和赋值都是在编译阶段完成的，没有运行时的参与。

3.1.4 小结 #
数组是 Go 语言中重要的数据结构，了解它的实现能够帮助我们更好地理解这门语言，通过对其实现的分析，我们知道了对数组的访问和赋值需要同时依赖编译器和运行时，它的大多数操作在编译期间都会转换成直接读写内存，在中间代码生成期间，编译器还会插入运行时方法 runtime.panicIndex 调用防止发生越界错误。

3.1.5 延伸阅读 #
Arrays, slices (and strings): The mechanics of ‘append’
Array vs Slice: accessing speed
