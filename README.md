# Preface

The general content of the project is not much different from the [DreamSoule's OLLVM16](https://github.com/DreamSoule/ollvm16) released before, only compatibility fixes were made for LLVM17. If you are interested, you can download the previous project and compare it to see the modified parts.

# Updates

Since Visual Studio cannot pass in annotations (maybe it's my fault?) I added a function to use function name matching to obfuscate whether this function is turned on or off

> Example:

```cpp
int function_fla_bcf_(int a);
```

# Legend

<details> 
<summary>Function source code:</summary>
<img src="./resource/fn_source.png"/>
</details>
<details> 
<summary>Original IDA decompilation</summary>
<img src="./resource/fn_ida.png"/>
</details>
<details> 
<summary>Add _fla_ to the function name</summary>
<img src="./resource/fn_ida_fla.png"/>
</details>
<details> 
<summary>Add _fla_bcf_ to the function name</summary>
<img src="./resource/fn_ida_fla_bcf.png"/>
</details>
</h7>

# Obfuscated feature list
> Command line add location: Project->Properties->C/C++->Command Line

```bash
- bcf # Bogus control flow
-   bcf_prob # Probability from 1% to 100%, defaults to 70
-   bcf_loop # Applies the Bogus control flow to the function the number of times specified, no restrictions, defaults to 2
- fla # Control Flow Flattening
- sub # Instruction substitution (add/and/sub/or/xor)
-   sub_loop # Number of instruction substitution, unlimited, defaults to 1
- sobf # String obfuscation (only narrow characters)
- split # Activates basic block splitting. Improve the flattening when applied together.
-   split_num # Applies the Bogus control flow to the function the number of times specified, no restrictions, defaults to 3
- ibr # Indirect Branching Pass
- icall # indirect calls (call register)
- igv # Indirect global variables
- fncmd # Enable function name control obfuscation ( function_fla_bcf_(); )
```

# All functions enabled
> Command line add location: Project->Properties->C/C++->Command Line<br>
> fla and bcf will cause the compilation speed to be very slow and some functions cannot be used<br>
> It is recommended to enable fncmd and remove -mllvm -fla -mllvm -bcf<br>
> Then add the corresponding control characters to the function names that need these two functions, for example: [#Update content]

```bash
-mllvm -fla -mllvm -bcf -mllvm -bcf_prob=80 -mllvm -bcf_loop=3 -mllvm -sobf -mllvm -icall -mllvm -ibr -mllvm -igv -mllvm -sub -mllvm -sub_loop=3 -mllvm -split -mllvm -split_num=5
```

# Official LLVM Patching Tutorial
1.Download LLVM official source code [LLVM 17.0.6](https://github.com/llvm/llvm-project/releases/tag/llvmorg-17.0.6) and unzip it<br>
2.Download this project and replace the files in the project with the official source code<br>
3.Use cmake to create the compilation tool generation file you need, taking VisualStudio 2022 as an example
```bash
cd llvm-project
mkdir build_vs2022
cd build_vs2022
cmake -G "Visual Studio 17 2022" -DCMAKE_C_FLAGS=/utf-8 -DCMAKE_CXX_FLAGS=/utf-8 -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_EH=OFF -DLLVM_ENABLE_RTTI=OFF -DLLVM_ENABLE_ASSERTIONS=ON -DLLVM_ENABLE_PROJECTS="clang;lld" -A x64 ../llvm
```
4.After cmake generates the solution, open LLVM.sln in the build_vs2022 directory and click Generate Solution.<br>
>If there is a prompt during the compilation process that some functions are private and cannot be called, just comment the private function. LLVM 17.0.6 works normally without error after commenting<br>
>To cancel the private attribute of the file and target line (LLVM17.0.1): ...\llvm-project\llvm\include\llvm\IR\Function.h:722<br>
>The private properties of this function: getBasicBlockList
# Patch details
> ...\llvm-project\llvm\lib\Passes\PassBuilder.cpp
```cpp
// Quote Obfuscation related documents
#include "Obfuscation/BogusControlFlow.h" // Bogus Control Flow
#include "Obfuscation/Flattening.h"  // Control flattening
#include "Obfuscation/SplitBasicBlock.h" // Basic block splitting
#include "Obfuscation/Substitution.h" // Instruction replacement
#include "Obfuscation/StringEncryption.h" // String encryption
#include "Obfuscation/IndirectGlobalVariable.h" // Indirect global variables
#include "Obfuscation/IndirectBranch.h" // Indirect branch pass
#include "Obfuscation/IndirectCall.h" // Indirect calls
#include "Obfuscation/Utils.h" // Functions utility (bool obf_function_name_cmd;)

// Add command line support
static cl::opt<bool> s_obf_split("split", cl::init(false), cl::desc("SplitBasicBlock: split_num=3(init)"));
static cl::opt<bool> s_obf_sobf("sobf", cl::init(false), cl::desc("String Obfuscation"));
static cl::opt<bool> s_obf_fla("fla", cl::init(false), cl::desc("Flattening"));
static cl::opt<bool> s_obf_sub("sub", cl::init(false), cl::desc("Substitution: sub_loop"));
static cl::opt<bool> s_obf_bcf("bcf", cl::init(false), cl::desc("BogusControlFlow: application number -bcf_loop=x must be x > 0"));
static cl::opt<bool> s_obf_ibr("ibr", cl::init(false), cl::desc("Indirect Branch"));
static cl::opt<bool> s_obf_igv("igv", cl::init(false), cl::desc("Indirect Global Variable"));
static cl::opt<bool> s_obf_icall("icall", cl::init(false), cl::desc("Indirect Call"));
static cl::opt<bool> s_obf_fn_name_cmd("fncmd", cl::init(false), cl::desc("use function name control obfuscation(_ + command + _ | example: function_fla_bcf_)"));

// Register the Pipeline callback directly in this function
PassBuilder::PassBuilder(...) {
...
  this->registerPipelineStartEPCallback(
      [](llvm::ModulePassManager &MPM,
         llvm::OptimizationLevel Level) {
        outs() << "[Soule] run.PipelineStartEPCallback\n";
        obf_function_name_cmd = s_obf_fn_name_cmd;
        if (obf_function_name_cmd) {
          outs() << "[Soule] enable function name control obfuscation(_ + command + _ | example: function_fla_)\n";
        }
        MPM.addPass(StringEncryptionPass(s_obf_sobf)); // Encrypt the string first. After the string encryption basic block appears, perform basic block segmentation and other obfuscation to increase the difficulty of decryption.
        llvm::FunctionPassManager FPM;
        FPM.addPass(IndirectCallPass(s_obf_icall)); // Indirect call pass
        FPM.addPass(SplitBasicBlockPass(s_obf_split)); // Basic block splitting pass
        FPM.addPass(FlatteningPass(s_obf_fla)); // Control flattening pass
        FPM.addPass(SubstitutionPass(s_obf_sub)); // Substitution pass
        FPM.addPass(BogusControlFlowPass(s_obf_bcf)); // Bogus control flow pass
        MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
        MPM.addPass(IndirectBranchPass(s_obf_ibr)); // Indirect instructions In theory, indirect instructions should be placed last.
        MPM.addPass(IndirectGlobalVariablePass(s_obf_igv)); // Indirect global variables
        MPM.addPass(RewriteSymbolPass()); // Rename specific symbols according to YAML information
      }
  );
}
```
> ...\llvm-project\llvm\lib\Passes\CMakeLists.txt
``` bash
# Add Obfuscation related source code
add_llvm_component_library(LLVMPasses
...
Obfuscation/Utils.cpp
Obfuscation/CryptoUtils.cpp
Obfuscation/ObfuscationOptions.cpp
Obfuscation/BogusControlFlow.cpp
Obfuscation/IPObfuscationContext.cpp
Obfuscation/Flattening.cpp
Obfuscation/StringEncryption.cpp
Obfuscation/SplitBasicBlock.cpp
Obfuscation/Substitution.cpp
Obfuscation/IndirectBranch.cpp
Obfuscation/IndirectCall.cpp
Obfuscation/IndirectGlobalVariable.cpp
...
)
```
# Credits
[LLVM](https://github.com/llvm/llvm-project)

[SsagePass](https://github.com/SsageParuders/SsagePass)

[wwh1004-ollvm16](https://github.com/wwh1004/ollvm-16)

[DreamSoule's ollvm17](https://github.com/DreamSoule/ollvm17)
