# SourcePawn compiler with custom features

This compiler is based on 1.10-dev branch of the official compiler. This is also one of my early experimental project, 1.11-dev version of the compiler has changed a lot, so this code is kind of outdated and I've decided to publish it.

## Features
### 1. Strip support 
By using `-s` in command line or `#pragma strip` in source would strip all the rtti and dgb sections. Basically remove all the names of your variables and functions(except those be declared as public). **But be aware that this would also break the runtime exception information.** This is useful when you are sure there won't be unexpected runtime exceptions and want to hide as much implementation detail as possible in your binary.
```sp
#pragma strip
#include <sourcemod>

public void OnPluginStart()
{
    ExceptionTest();
}

void ExceptionTest()
{
    int index = 4;
    int array[4];
    
    //index out of bounds exception
    array[index] = 4;
}
```
The console output of the exception.
```
Exception reported: Array index out-of-bounds (index 4, limit 4)
Blaming: plugins/test.smx
Call stack trace:
  [1] Line 0, <unknown>::sub_BFC
  [2] Line 0, <unknown>::OnPluginStart
```
  
### 2. Anti-decompiler support
By using `-P` in command line or `#pragma protection` in source would break the lysis decompiler. **Note that this "protection" is vulnerable**, the implementation is simply trigger an unhandled runtime exception inside the decompiler to interrupt the analysis of each funtions. So it's easy to fix, it's just a matter of time. 
  
I've been using this feature for a long time myself, now I don't need this so it's published and maybe it will be dead soon, think twice before you chose to use this to protect your binary. Maybe I will update another approach when this was fixed, but your binary would already be naked by then.
```sp
#pragma protection
```
For those who want to find their own stable way to implement this, I'll shortly explain my implementation to make an example. The following asm code is a comparison between normal function template and function template when using #pragma protection.   

Normal function template.
```asm
CODE 0	
	proc	
	; ... Your code here
	; ......
	retn
```
Function template when using #pragma protection.
```asm
CODE 0	
	proc	
	jump 105
	sysreq.n 0xdeadbeef 0
l.105	
	; ... Your code here
	; ......
	retn
```

Opcode `sysreq.n` is used everytime we call a native function, and each smx binary contains a native table in the `.natives` section. The first oprand is the position index of the native function in the table, the second is the actual argument passed to that native. So it's obvious that there is no way a smx binary would use so many (0xdeadbeef) native functions, and accidentally the decompiler also thinks so.   
  
In `/src/main/java/lysis/builder/MethodParser.java` (Decompiler source code)
```java
private LInstruction readInstruction(SPOpcode op) throws Exception {
	switch (op) {
		//case ....
		case sysreq_n: {
			long index = readUInt32();   //Read the native table index, which is 0xdeadbeef
			add(new LPushConstant((int) readUInt32()));  //Read number of arguments
			return new LSysReq(file_.natives()[(int) index]);  //Index out of bounds exception
		}
		//case ....
	}
}
```

When analyse each function there will be index out of bounds exception, which is handled very top level so the rest of the code won't be analysed, that's how to "protect" our binary, also explains why this kind of protection is vulnerable.
 
Every function in the decompiler output should look like this:
```
/* ERROR PREPROCESSING! Index -559038737 out of bounds for length 2 */
 function "OnPluginStart" (number 0)
 
 /* ERROR PREPROCESSING! Index -559038737 out of bounds for length 2 */
 function "__ext_core_SetNTVOptional" (number 1)
```

This instruction should not be executed in the vm, or there will be exception either. So we use `jump` opcode to never execute that instruction. Now the job is done, a fully functioning smx binary with protection against the decompiler. The key is to check through the decompiler's source to find out where to cause proper exception.

### 3. #pragma warning support
#pragma warning allows you to treat some warnings as errors or disable some warnings. **Note that this derective is globally scoped.** The valid operation is disable or error.
```sp
#pragma warning(disable : 204)       //symbol is assigned a value that is never used: "%s"
#pragma warning(error : 213)         //tag mismatch

#include <sourcemod>

public void OnPluginStart()
{
    //Test warning 204, now it should't trigger warning
    int index_nowarning = 36;

    //Test warning 213, this would now trigger an error.
    float float_trigger_error = 32;
}

```
Compiler output.
```
test.sp(12) : error 213: tag mismatch
```

### 4. Build-in macro `__SP_COMPILER_STRIP_AND_PROTECTION__`
This allows you to safely use these features in your project. Wrap all the new derectives in the following block makes it possible for your souces to be compiled in any other official compilers.
```sp
#if defined __SP_COMPILER_STRIP_AND_PROTECTION__
   #pragma strip
   #pragma protection
   #pragma warning(disable : 204)
#endif
```
