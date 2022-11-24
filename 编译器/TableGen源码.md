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

## EmitInstrInfo

```cpp
void EmitInstrInfo(RecordKeeper &RK, raw_ostream &OS) {
  InstrInfoEmitter(RK).run(OS);
  EmitMapTable(RK, OS);
}
```

## run

```cpp
// run - Emit the main instruction description records for the target...
void InstrInfoEmitter::run(raw_ostream &OS) {
  emitSourceFileHeader("Target Instruction Enum Values and Descriptors", OS);
  emitEnums(OS);

  OS << "#ifdef GET_INSTRINFO_MC_DESC\n";
  OS << "#undef GET_INSTRINFO_MC_DESC\n";

  OS << "namespace llvm {\n\n";

  CodeGenTarget &Target = CDP.getTargetInfo();
  const std::string &TargetName = Target.getName();
  Record *InstrInfo = Target.getInstructionSet();

  // Keep track of all of the def lists we have emitted already.
  std::map<std::vector<Record*>, unsigned> EmittedLists;
  unsigned ListNumber = 0;

  // Emit all of the instruction's implicit uses and defs.
  for (const CodeGenInstruction *II : Target.getInstructionsByEnumValue()) {
    Record *Inst = II->TheDef;
    std::vector<Record*> Uses = Inst->getValueAsListOfDefs("Uses");
    if (!Uses.empty()) {
      unsigned &IL = EmittedLists[Uses];
      if (!IL) PrintDefList(Uses, IL = ++ListNumber, OS);
    }
    std::vector<Record*> Defs = Inst->getValueAsListOfDefs("Defs");
    if (!Defs.empty()) {
      unsigned &IL = EmittedLists[Defs];
      if (!IL) PrintDefList(Defs, IL = ++ListNumber, OS);
    }
  }

  OperandInfoMapTy OperandInfoIDs;

  // Emit all of the operand info records.
  EmitOperandInfo(OS, OperandInfoIDs);

  // Emit all of the MCInstrDesc records in their ENUM ordering.
  //
  OS << "\nextern const MCInstrDesc " << TargetName << "Insts[] = {\n";
  ArrayRef<const CodeGenInstruction*> NumberedInstructions =
    Target.getInstructionsByEnumValue();

  SequenceToOffsetTable<std::string> InstrNames;
  unsigned Num = 0;
  for (const CodeGenInstruction *Inst : NumberedInstructions) {
    // Keep a list of the instruction names.
    InstrNames.add(Inst->TheDef->getName());
    // Emit the record into the table.
    emitRecord(*Inst, Num++, InstrInfo, EmittedLists, OperandInfoIDs, OS);
  }
  OS << "};\n\n";

  // Emit the array of instruction names.
  InstrNames.layout();
  OS << "extern const char " << TargetName << "InstrNameData[] = {\n";
  InstrNames.emit(OS, printChar);
  OS << "};\n\n";

  OS << "extern const unsigned " << TargetName <<"InstrNameIndices[] = {";
  Num = 0;
  for (const CodeGenInstruction *Inst : NumberedInstructions) {
    // Newline every eight entries.
    if (Num % 8 == 0)
      OS << "\n    ";
    OS << InstrNames.get(Inst->TheDef->getName()) << "U, ";
    ++Num;
  }

  OS << "\n};\n\n";

  // MCInstrInfo initialization routine.
  OS << "static inline void Init" << TargetName
     << "MCInstrInfo(MCInstrInfo *II) {\n";
  OS << "  II->InitMCInstrInfo(" << TargetName << "Insts, "
     << TargetName << "InstrNameIndices, " << TargetName << "InstrNameData, "
     << NumberedInstructions.size() << ");\n}\n\n";

  OS << "} // end llvm namespace\n";

  OS << "#endif // GET_INSTRINFO_MC_DESC\n\n";

  // Create a TargetInstrInfo subclass to hide the MC layer initialization.
  OS << "#ifdef GET_INSTRINFO_HEADER\n";
  OS << "#undef GET_INSTRINFO_HEADER\n";

  std::string ClassName = TargetName + "GenInstrInfo";
  OS << "namespace llvm {\n";
  OS << "struct " << ClassName << " : public TargetInstrInfo {\n"
     << "  explicit " << ClassName
     << "(int CFSetupOpcode = -1, int CFDestroyOpcode = -1, int CatchRetOpcode = -1, int ReturnOpcode = -1);\n"
     << "  ~" << ClassName << "() override = default;\n";


  OS << "\n};\n} // end llvm namespace\n";

  OS << "#endif // GET_INSTRINFO_HEADER\n\n";

  OS << "#ifdef GET_INSTRINFO_HELPER_DECLS\n";
  OS << "#undef GET_INSTRINFO_HELPER_DECLS\n\n";
  emitTIIHelperMethods(OS, TargetName, /* ExpandDefintion = */false);
  OS << "\n";
  OS << "#endif // GET_INSTRINFO_HELPER_DECLS\n\n";

  OS << "#ifdef GET_INSTRINFO_HELPERS\n";
  OS << "#undef GET_INSTRINFO_HELPERS\n\n";
  emitTIIHelperMethods(OS, TargetName, /* ExpandDefintion = */true);
  OS << "#endif // GET_INSTRINFO_HELPERS\n\n";

  OS << "#ifdef GET_INSTRINFO_CTOR_DTOR\n";
  OS << "#undef GET_INSTRINFO_CTOR_DTOR\n";

  OS << "namespace llvm {\n";
  OS << "extern const MCInstrDesc " << TargetName << "Insts[];\n";
  OS << "extern const unsigned " << TargetName << "InstrNameIndices[];\n";
  OS << "extern const char " << TargetName << "InstrNameData[];\n";
  OS << ClassName << "::" << ClassName
     << "(int CFSetupOpcode, int CFDestroyOpcode, int CatchRetOpcode, int ReturnOpcode)\n"
     << "  : TargetInstrInfo(CFSetupOpcode, CFDestroyOpcode, CatchRetOpcode, ReturnOpcode) {\n"
     << "  InitMCInstrInfo(" << TargetName << "Insts, " << TargetName
     << "InstrNameIndices, " << TargetName << "InstrNameData, "
     << NumberedInstructions.size() << ");\n}\n";
  OS << "} // end llvm namespace\n";

  OS << "#endif // GET_INSTRINFO_CTOR_DTOR\n\n";

  emitOperandNameMappings(OS, Target, NumberedInstructions);

  emitOperandTypeMappings(OS, Target, NumberedInstructions);

  emitMCIIHelperMethods(OS, TargetName);
}
```

## emitEnums

```cpp
void InstrInfoEmitter::emitEnums(raw_ostream &OS) {
  OS << "#ifdef GET_INSTRINFO_ENUM\n";
  OS << "#undef GET_INSTRINFO_ENUM\n";

  OS << "namespace llvm {\n\n";

  CodeGenTarget Target(Records);

  // We must emit the PHI opcode first...
  StringRef Namespace = Target.getInstNamespace();

  if (Namespace.empty())
    PrintFatalError("No instructions defined!");

  OS << "namespace " << Namespace << " {\n";
  OS << "  enum {\n";
  unsigned Num = 0;
  for (const CodeGenInstruction *Inst : Target.getInstructionsByEnumValue())
    OS << "    " << Inst->TheDef->getName() << "\t= " << Num++ << ",\n";
  OS << "    INSTRUCTION_LIST_END = " << Num << "\n";
  OS << "  };\n\n";
  OS << "} // end " << Namespace << " namespace\n";
  OS << "} // end llvm namespace\n";
  OS << "#endif // GET_INSTRINFO_ENUM\n\n";

  OS << "#ifdef GET_INSTRINFO_SCHED_ENUM\n";
  OS << "#undef GET_INSTRINFO_SCHED_ENUM\n";
  OS << "namespace llvm {\n\n";
  OS << "namespace " << Namespace << " {\n";
  OS << "namespace Sched {\n";
  OS << "  enum {\n";
  Num = 0;
  for (const auto &Class : SchedModels.explicit_classes())
    OS << "    " << Class.Name << "\t= " << Num++ << ",\n";
  OS << "    SCHED_LIST_END = " << Num << "\n";
  OS << "  };\n";
  OS << "} // end Sched namespace\n";
  OS << "} // end " << Namespace << " namespace\n";
  OS << "} // end llvm namespace\n";

  OS << "#endif // GET_INSTRINFO_SCHED_ENUM\n\n";
}
```

--gen-instr-info 后端

一、排序并输出每条指令的枚举值

Target.getInstructionsByEnumValue（）

返回目标定义的所有指令，按其枚举值排序。
还保证以下指令顺序：
-include/llvm/Support/TargetOpcodes.def中声明的固定/通用指令，按顺序；
-按名称排序的词典顺序的伪指令；Sw64InstrInfo.td中所有继承PseudoInstSw64的记录就是伪指令。
-按名称排序的词典顺序的其他指令。与架构有关的指令

二、将指令分类，并输出枚举值

CodeGenSchedClass注释：

调度类。
每个指令描述都将映射到调度类。有四种类型的类：
1） 设置了ItinClassDef的显式定义的行程类。Writes和ReadDefs为空。对任意处理器ProcIndices都是0（表示适用于所有的处理器）。
2） 一种隐含类，包含指令定义中定义的SchedWrites和SchedReads列表，它们在所有子目标中都是通用的。对任意处理器ProcIndices都是0（表示适用于所有的处理器）。
3） 一个隐含类，包含一系列InstRW记录，这些记录将指令映射到每个处理器的SchedWrites和SchedReads。InstrClassMap应将相同的指令映射到此类。ProcIndices包含为该类提供InstrRW记录的所有处理器。对于没有InstRW条目的处理器，仍可以定义ItinClassDef或写入/读取。
4） 推断的类表示可以在运行时解析的另一类的变体。ProcIndices包含可能需要该类的一组处理器。随着变量的扩展，ProcIndex通过SchedClasses传播。可以从一个行程类别推断出多个SchedClasses。每个都从ItinRW记录中继承处理器索引，该记录将行程类映射到变量Writes或Reads

第二种情况

include/llvm/Target/TargetSchedule.td class InstRW

将一组操作码映射到SchedReadWrite类型列表。这允许子目标轻松覆盖特定操作。SchedModel将此操作码映射绑定到处理器.

/lib/Target/Sw64/Sw64SchedCore3.td中继承InstRW的匿名记录的第二个模板参数就是指令列表

三、输出执行指令时隐式使用（Uses）和定义（Defs）的寄存器列表。

会获取指令记录中的Uses字段与Defs字段，如果有的话输出

四、操作数信息列表

-1表示操作数没有固定寄存器类。

0表示适用的标志。

```cpp
class MCOperandInfo {
public:
  /// This specifies the register class enumeration of the operand
  /// if the operand is a register.  If isLookupPtrRegClass is set, then this is
  /// an index that is passed to TargetRegisterInfo::getPointerRegClass(x) to
  /// get a dynamic register class.
  //如果操作数是寄存器，则指定操作数的寄存器类枚举。
  //如果设置了isLookupPtrRegClass，则这是传递给
  //TargetRegisterInfo:：getPointerRegClass（x）以获取动态寄存器类的索引。
  int16_t RegClass;

  /// These are flags from the MCOI::OperandFlags enum.
  uint8_t Flags;

  /// Information about the type of the operand.
  uint8_t OperandType;
  /// The lower 16 bits are used to specify which constraints are set.
  /// The higher 16 bits are used to specify the value of constraints (4 bits
  /// each).
  uint32_t Constraints;

  /// Set if this operand is a pointer value and it requires a callback
  /// to look up its register class.
  //设置此操作数是否为指针值，并且需要回调来查找其寄存器类。
  bool isLookupPtrRegClass() const {
    return Flags & (1 << MCOI::LookupPtrRegClass);
  }

  /// Set if this is one of the operands that made up of the predicate
  /// operand that controls an isPredicable() instruction.
  //设置这是否是由控制isPredical（）指令的谓词操作数组成的操作数之一。
  bool isPredicate() const { return Flags & (1 << MCOI::Predicate); }

  /// Set if this operand is a optional def.
  bool isOptionalDef() const { return Flags & (1 << MCOI::OptionalDef); }

  bool isGenericType() const {
    return OperandType >= MCOI::OPERAND_FIRST_GENERIC &&
           OperandType <= MCOI::OPERAND_LAST_GENERIC;
  }

  unsigned getGenericTypeIndex() const {
    assert(isGenericType() && "non-generic types don't have an index");
    return OperandType - MCOI::OPERAND_FIRST_GENERIC;
  }
};
```
