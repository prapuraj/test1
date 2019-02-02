#Fuzz Windows Via Javascript

Microsoft in February and March patched my two win32k reported vulnerability in the end of last year MS16-018 / CVE-2016-0048, M16-034 / CVE-2016-0096,

Details about these two vulnerabilities, who are interested can look at PatchDiff, do not write this today.

For both vulnerabilities behind the story, but added a bit mean.

in the fuzz process, frequent changes fuzzer code, and verify that the idea is a very common procedure.

In order to ensure a clean, fuzz virtual machine environment where there is no third-party software installed. and so

After each change in the physical machine in fuzzer code, then compiled, and then transferred to the virtual machine, is a cumbersome and inefficient thing.

When beginning fuzzer are written in C, so in the virtual machine also installed a set of WDK build environment, then you need to verify every idea in the virtual machine, 
use Notepad ++ directly modify fuzzer code, WDK compile, and then execute the compiled bin.


In early August last year, inadvertently discovered this js Microsoft's Chakra engine,

Since the V8 can be used for nodejs, auxiliary back-end development.

Why not use js to write fuzzer, and the syntax is very close to C, and presumably it will be very easy to modify the code before the transplant write c fuzzer will be very easy.

began to study the Chakra's document, which is later JSRT project, meaning that Javascript Runtime,


Chakra itself is just a basic js engine, can only perform basic expressions.

In the MS documentation for use js just want to be as a plug-language software to the SDK functions required to provide each one write a wrapper function after registration with JsCreateFunction, js can call.


But my goal is to use js to Fuzz windows kernel, faced with the entire operating system, win32 api function too much, it is impossible to write a c each function wrapper function to call the js.

So think of some tips to hack this process.

Since js no pointer types, and many api functions are required to pass a parameter of type pointer, which need to have the ability to direct js memory for the pointer itself, to simulate the use of js Lane integer.

Use c to provide malloc / free, getCHAR ... setULONG, and other functions, js will be able to directly read and write memory.

In order to obtain the address of the function, the module must first know the address, so I offered LoadLibrary, GetModuleHandle

Then GetProcAddress help, you can get the address of any function.

Have the ability to directly read and write memory, plus the address of the function to obtain, you can directly call the api.


For example, the number of parameters for the two types of stdcall functions can be abstract,

RoutineResult = ((LPFN_STDCALL2Param) (RoutineAddress)) (ArgArray [0], ArgArray [1]);

Get the return value type is DWORD in js layer, and then manually type can be converted to js.


For the parameters relatively simple functions, such as Beep, parameter types are numeric, handled well.

But for EnumWindow this, we need to back off a function as a parameter.

We need to complete js-> c -> js cross-language process.


For js-> c is better understood, but c -> js this process a little trouble.

Because c-> js process, in addition to the required parameters js, Charka have passed the required information such as the current Context.

Then think of the idea of using Thunk, use asm to re-layout of the parameters on the stack, Runtime Context additional information transfer Chakra needs, current js object information, the callback function and js objects.


Thus, in the C level, provide only minimal auxiliary Chakra, can make js direct cross-language calls all api.

Facts have proved that this JSRT still very convenient.

For example, when calling ShadowSSDT in function prototypes need to define the parameters used directly

function NtGdiAngleArc (arg_01, arg_02, arg_03, arg_04, arg_05, arg_06)
{

	return Invoke (W32pServiceTable [ "NtGdiAngleArc"], arg_01, arg_02, arg_03, arg_04, arg_05, arg_06);
}

SDT can be convenient to call any function.


There were not enough light, you also need a js library to assist the basic call, such as js does not have the printf function, and so on.

Because of frequent maintenance service module is interrupted during the write off of a few months.


In early November last year, because of a game, switch targets to win32k, just to verify the effectiveness and convenience of this environment.

So with the later CVE-2016-0096, it may be the first to use js fuzz out of the core hole.

When the blue screen, you can see a very interesting stack from jscript all the way to the core.

![](./jsfuzz.jpg)