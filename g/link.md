没头绪，先记这


cmd/internal/obj/link.go 貌似是承前启后的地方，很多关心的工作应该是在这里干的。

```
// An Addr is an argument to an instruction.
// The general forms and their encodings are:
//
//	sym±offset(symkind)(reg)(index*scale)
//		Memory reference at address &sym(symkind) + offset + reg + index*scale.
//		Any of sym(symkind), ±offset, (reg), (index*scale), and *scale can be omitted.
//		If (reg) and *scale are both omitted, the resulting expression (index) is parsed as (reg).
//		To force a parsing as index*scale, write (index*1).
//		Encoding:
//			type = TYPE_MEM
//			name = symkind (NAME_AUTO, ...) or 0 (NAME_NONE)
//			sym = sym
//			offset = ±offset
//			reg = reg (REG_*)
//			index = index (REG_*)
//			scale = scale (1, 2, 4, 8)
//
type Addr struct {
	Type   int16
	Reg    int16
	Index  int16
	Scale  int16 // Sometimes holds a register.
	Name   int8
	Class  int8
	Etype  uint8
	Offset int64
	Width  int64
	Sym    *LSym
	Gotype *LSym

	// argument value:
	//	for TYPE_SCONST, a string
	//	for TYPE_FCONST, a float64
	//	for TYPE_BRANCH, a *Prog (optional)
	//	for TYPE_TEXTSIZE, an int32 (optional)
	Val interface{}

	Node interface{} // for use by compiler
}

```

```
// An LSym is the sort of symbol that is written to an object file.
type LSym struct {
	Name      string
	Type      int16
	Version   int16
	Dupok     uint8
	Cfunc     uint8
	Nosplit   uint8
	Leaf      uint8
	Seenglobl uint8
	Onlist    uint8
	// Local means make the symbol local even when compiling Go code to reference Go
	// symbols in other shared libraries, as in this mode symbols are global by
	// default. "local" here means in the sense of the dynamic linker, i.e. not
	// visible outside of the module (shared library or executable) that contains its
	// definition. (When not compiling to support Go shared libraries, all symbols are
	// local in this sense unless there is a cgo_export_* directive).
	Local  bool
	Args   int32
	Locals int32
	Value  int64
	Size   int64
	Next   *LSym
	Gotype *LSym
	Autom  *Auto
	Text   *Prog
	Etext  *Prog
	Pcln   *Pcln
	P      []byte
	R      []Reloc
}


const (
	R_TLS
	// R_TLS_LE (only used on 386 and amd64 currently) resolves to the offset of the
	// thread-local g from the thread local base and is used to implement the "local
	// exec" model for tls access (r.Sym is not set by the compiler for this case but
	// is set to Tlsg in the linker when externally linking).
	R_TLS_LE
	// R_TLS_IE (only used on 386 and amd64 currently) resolves to the PC-relative
	// offset to a GOT slot containing the offset the thread-local g from the thread
	// local base and is used to implemented the "initial exec" model for tls access
	// (r.Sym is not set by the compiler for this case but is set to Tlsg in the
	// linker when externally linking).

)
	
	
	
type Pcdata struct {
	P []byte
}


// Link holds the context for writing object code from a compiler
// to be linker input or for reading that input into the linker.
type Link struct {
	Arch               *LinkArch
}

// LinkArch is the definition of a single architecture.
type LinkArch struct {
	ByteOrder  binary.ByteOrder
	Name       string
	Thechar    int
	Preprocess func(*Link, *LSym)
	Assemble   func(*Link, *LSym)
	Follow     func(*Link, *LSym)
	Progedit   func(*Link, *Prog)
	UnaryDst   map[int]bool // Instruction takes one operand, a destination.
	Minlc      int
	Ptrsize    int
	Regsize    int
}

type Plist struct {
	Name    *LSym
	Firstpc *Prog
	Recur   int
	Link    *Plist
}

```


