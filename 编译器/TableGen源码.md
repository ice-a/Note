# 分析过程

# 词法分析器TGLexer

```cpp
class TGLexer {
  SourceMgr &SrcMgr;

  const char *CurPtr;//当前分析器分析位置
  StringRef CurBuf;

  const char *TokStart;//当前记号的起始位置
  tgtok::TokKind CurCode;//返回记号
  std::string CurStrVal;  
  int64_t CurIntVal;     

  unsigned CurBuffer;
}
```

# 语法分析器TGParser

```cpp
class TGParser {
  TGLexer Lex;
  std::vector<SmallVector<LetRecord, 4>> LetStack;
  std::map<std::string, std::unique_ptr<MultiClass>> MultiClasses;

  std::vector<std::unique_ptr<ForeachLoop>> Loops;

  SmallVector<DefsetRecord *, 2> Defsets;

  MultiClass *CurMultiClass;

  RecordKeeper &Records;

  enum IDParseMode {
    ParseValueMode,   
    ParseNameMode,    

  };
  tgtok::TokKind Lex() {//TokKind枚举类型，包含了记号的所有种类
    return CurCode = LexToken(CurPtr == CurBuf.begin());
  }
  private:

  tgtok::TokKind LexToken(bool FileOrLineStart = false);
}
```

# RecordKeeper

RecordKeeper 类的实例充当 TableGen 解析和收集的所有类和记录的容器。当 TableGen 调用 RecordKeeper 实例时，它将被传递到后端。这个类通常缩写为 RK。

在记录保管员中有两个映射，一个用于类，一个用于记录(后者通常称为 defs)。每个映射都将类或记录名映射到 Record 类的一个实例(参见 Record) ，该实例包含关于该类或记录的所有信息。

```cpp
class RecordKeeper {
  friend class RecordRecTy;
  using RecordMap = std::map<std::string, std::unique_ptr<Record>>;
  RecordMap Classes, Defs;//一个类映射、一个记录映射
  FoldingSet<RecordRecTy> RecordTypePool;
  std::map<std::string, Init *> ExtraGlobals;
  unsigned AnonCounter = 0;
}
```

# Record

```cpp
class Record {
  static unsigned LastID;

  Init *Name;
  // Location where record was instantiated, followed by the location of
  // multiclass prototypes used.
  SmallVector<SMLoc, 4> Locs;
  SmallVector<Init *, 0> TemplateArgs;//对于类记录，为类的模板参数的向量。
  SmallVector<RecordVal, 0> Values;//字段名和值

  // All superclasses in the inheritance forest in reverse preorder (yes, it
  // must be a forest; diamond-shaped inheritance is not allowed).
  SmallVector<std::pair<Record *, SMRange>, 0> SuperClasses;

  // Tracks Record instances. Not owned by Record.
  RecordKeeper &TrackedRecords;

  DefInit *TheInit = nullptr;

  // Unique record ID.
  unsigned ID;//一个唯一的记录 ID

  bool IsAnonymous;//指定此记录是否为匿名记录的布尔值。
  bool IsClass;//一个布尔值，指定这是否是类定义。

  void checkName();
}
```

# RecordVal

```cpp
class RecordVal {
  friend class Record;

  Init *Name;//字段名称
  PointerIntPair<RecTy *, 1, bool> TyAndPrefix;//类型
  Init *Value;//字段值
}
```

# main()

```cpp
int main(int argc, char **argv) {
  sys::PrintStackTraceOnErrorSignal(argv[0]);
  PrettyStackTraceProgram X(argc, argv);
  cl::ParseCommandLineOptions(argc, argv);

  llvm_shutdown_obj Y;

  return TableGenMain(argv[0], &LLVMTableGenMain);//
}
```

# TableGenMain()

```cpp
int llvm::TableGenMain(char *argv0, TableGenMainFn *MainFn) {
  RecordKeeper Records;

  //分析输入文件
  ErrorOr<std::unique_ptr<MemoryBuffer>> FileOrErr =
      MemoryBuffer::getFileOrSTDIN(InputFilename);
  if (std::error_code EC = FileOrErr.getError())//输入错误
    return reportError(argv0, "Could not open input file '" + InputFilename +
                                  "': " + EC.message() + "\n");

  //告诉SrcMgr这个缓冲区，这是TGParser将获取的。
  SrcMgr.AddNewSourceBuffer(std::move(*FileOrErr), SMLoc());//初始化SrcMgr


  //记录include目录的位置，以便lexer稍后可以找到它。
  SrcMgr.setIncludeDirs(IncludeDirs);//IncludeDirs -I 参数
  //TGParser的构造函数，实例化分析器TGParser，同时会调用TGLexer构造器。
  TGParser Parser(SrcMgr, MacroNames, Records);//MacroNames -D 参数
  //进行词法、语法分析。
  if (Parser.ParseFile())
    return 1;

  //将输出写入内存。
  std::string OutString;
  raw_string_ostream Out(OutString);//输出流
  if (MainFn(Out, Records))//LLVMTableGenMain 选择TableGen后端
    return 1;

  //始终写入depfile，即使主输出没有更改。如果它丢失了，Ninja认为输出是脏的。
  //如果这低于下面的早期退出，并且有人删除了.inc.d文件，但没有删除.inc文件，tablegen将永远不会写入DEP文件。
  if (!DependFilename.empty()) {
    if (int Ret = createDependencyFile(Parser, argv0))
      return Ret;
  }

  //如果存在任何差异，则仅更新实际输出文件。这可以防止重新编译依赖于它的所有文件（如果没有）。
  if (auto ExistingOrErr = MemoryBuffer::getFile(OutputFilename))
    if (std::move(ExistingOrErr.get())->getBuffer() == Out.str())
      return 0;

  std::error_code EC;
  ToolOutputFile OutFile(OutputFilename, EC, sys::fs::F_Text);
  if (EC)//输出错误
    return reportError(argv0, "error opening " + OutputFilename + ":" +
                                  EC.message() + "\n");
  OutFile.os() << Out.str();//输出信息

  if (ErrorsPrinted > 0)
    return reportError(argv0, Twine(ErrorsPrinted) + " errors.\n");

  // Declare success.
  OutFile.keep();
  return 0;
}
```

## TGParser实例化

```cpp
TGLexer::TGLexer(SourceMgr &SM, ArrayRef<std::string> Macros) : SrcMgr(SM) {
  CurBuffer = SrcMgr.getMainFileID();
  CurBuf = SrcMgr.getMemoryBuffer(CurBuffer)->getBuffer();
  CurPtr = CurBuf.begin();
  TokStart = nullptr;

  PrepIncludeStack.push_back(
      make_unique<std::vector<PreprocessorControlDesc>>());

  std::for_each(Macros.begin(), Macros.end(),
                [this](const std::string &MacroName) {
                  DefinedMacros.insert(MacroName);
                });
}
```

## TGParser::ParseFile()

```cpp
bool TGParser::ParseFile() {
  Lex.Lex(); // Prime the lexer. //启动lexer 词法分析器lexer
  if (ParseObjectList()) return true;//语法分析器Parser

  if (Lex.getCode() == tgtok::Eof)
    return false;

  return TokError("Unexpected input at top level");
}
```

# 输出过程

## LLVMTableGenMain

```cpp
bool LLVMTableGenMain(raw_ostream &OS, RecordKeeper &Records) {
  switch (Action) {
  case PrintRecords:
    OS << Records;           // No argument, dump all contents
    break;
  case DumpJSON:
    EmitJSON(Records, OS);
    break;
  case GenEmitter:
    EmitCodeEmitter(Records, OS);
    break;
  case GenRegisterInfo:
    EmitRegisterInfo(Records, OS);
    break;
  case GenInstrInfo:
    EmitInstrInfo(Records, OS);
    break;
  case GenInstrDocs:
    EmitInstrDocs(Records, OS);
    break;
  case GenCallingConv:
    EmitCallingConv(Records, OS);
    break;
  case GenAsmWriter:
    EmitAsmWriter(Records, OS);
    break;
  case GenAsmMatcher:
    EmitAsmMatcher(Records, OS);
    break;
  case GenDisassembler:
    EmitDisassembler(Records, OS);
    break;
  case GenPseudoLowering:
    EmitPseudoLowering(Records, OS);
    break;
  case GenCompressInst:
    EmitCompressInst(Records, OS);
    break;
  case GenDAGISel:
    EmitDAGISel(Records, OS);
    break;
  case GenDFAPacketizer:
    EmitDFAPacketizer(Records, OS);
    break;
  case GenFastISel:
    EmitFastISel(Records, OS);
    break;
  case GenSubtarget:
    EmitSubtarget(Records, OS);
    break;
  case GenIntrinsicEnums:
    EmitIntrinsicEnums(Records, OS);
    break;
  case GenIntrinsicImpl:
    EmitIntrinsicImpl(Records, OS);
    break;
  case GenTgtIntrinsicEnums:
    EmitIntrinsicEnums(Records, OS, true);
    break;
  case GenTgtIntrinsicImpl:
    EmitIntrinsicImpl(Records, OS, true);
    break;
  case GenOptParserDefs:
    EmitOptParser(Records, OS);
    break;
  case PrintEnums:
  {
    for (Record *Rec : Records.getAllDerivedDefinitions(Class))
      OS << Rec->getName() << ", ";
    OS << "\n";
    break;
  }
  case PrintSets:
  {
    SetTheory Sets;
    Sets.addFieldExpander("Set", "Elements");
    for (Record *Rec : Records.getAllDerivedDefinitions("Set")) {
      OS << Rec->getName() << " = [";
      const std::vector<Record*> *Elts = Sets.expand(Rec);
      assert(Elts && "Couldn't expand Set instance");
      for (Record *Elt : *Elts)
        OS << ' ' << Elt->getName();
      OS << " ]\n";
    }
    break;
  }
  case GenCTags:
    EmitCTags(Records, OS);
    break;
  case GenAttributes:
    EmitAttributes(Records, OS);
    break;
  case GenSearchableTables:
    EmitSearchableTables(Records, OS);
    break;
  case GenGlobalISel:
    EmitGlobalISel(Records, OS);
    break;
  case GenRegisterBank:
    EmitRegisterBank(Records, OS);
    break;
  case GenX86EVEX2VEXTables:
    EmitX86EVEX2VEXTables(Records, OS);
    break;
  case GenX86FoldTables:
    EmitX86FoldTables(Records, OS);
    break;
  case GenExegesis:
    EmitExegesis(Records, OS);
    break;
  }

  return false;
}
}
```
