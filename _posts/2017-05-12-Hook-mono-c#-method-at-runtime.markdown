---
layout: post
title:  "Hook mono c# method at runtime (运行过程中hook mono C#函数)"
date:   2017-05-12 17:04:01 +0800
categories: Tech C#
---

最近有一个需求：针对基于mono框架的程序，在C#层面实现函数hook。网上搜罗一番，并没有找到现成可用的工具，故自己调研,  整理于此。

## 思路与原理

C#代码实际上是被编译成了中间语言CIL(Common Intermediate Language,CIL), CIL在运行时会被mono虚拟机解释为机器语言来执行。所以，一个显而易见的思路是，先hook mono虚拟机中CIL解释器，然后通过修改C#函数所对应的CIL来实现hook。


## 前提条件和准备工作

1. 必须具备系统管理员权限。例如，linux或者android具备root权限，windows具有管理员权限。（由于ios平台本身不支持即时编译，所以并不适用于此方法）。

2. 已经具备native层的Hook能力。例如，linux或者android平台利用ptrace进行so的远程注入，windows平台利用easyhook工具进行dll的远程注入。

## 实现

下面我以windows平台为例来介绍如何实现mono C# hook, 本文假定读者已经了解了如何使用easyhook （不太了解easyhook的读者可以自行调研下，很简单的一个hook工具）。有Android和linux平台hook需求的读者可以参考此文原理自行实现，只是底层hook方法换成ptrace而已。

首先，向目标进程注入我们编写的dll。
```c
int _tmain(int argc, _TCHAR* argv[])
{
  ......
  WCHAR* dllToInject = L"HookDll.dll";
  wprintf(L"Attempting to inject: %s\n\n", dllToInject);
  // Inject dllToInject into the target process Id, passing
  // freqOffset as the pass through data.
  NTSTATUS nt = RhInjectLibrary(
  processId, // The process to inject into
  0, // ThreadId to wake up upon injection
  EASYHOOK_INJECT_DEFAULT,
  dllToInject, // 32-bit
  dllToInject, // 64-bit not provided
  0, // data to send to injected DLL entry point
  0// size of data to send
  );
  .......
}

```
然后，目标进程会执行到我们注入的dll的程序入口NativeInjectionEntryPoint， 我们先获取mono虚拟机的mono.dll在目标进程的基地址。
```c
void __stdcall NativeInjectionEntryPoint(REMOTE_ENTRY_INFO* inRemoteInfo)
{
    HMODULE address = GetModuleHandle(TEXT("mono.dll"));
    ...........
```
接着，我们回到之前讨论的思路中来：“要想办法hook到mono.dll中的编译函数，继而在这些函数内存（或者说是参数）中获取到C#层函数的字节码并将其修改”。那么应该hook哪个编译器函数呢？由于即时编译一般都会有缓存，每个上层C#函数被成功编译成机器语言后会被设置成“已编译”的状态，后续该C#函数就不会再被mono编译。所以如果在目标程序已经跑起来的情况下，我们只是随便Hook一个mono编译函数，有可能一次都不会被触发调用。

事实上，mono虚拟机不止负责编译上层函数字节码，也负责调用执行编译后的机器代码。所以一个可行的解决方案是：hook mono的jit调用函数，而不是编译函数，这样能保证我们无论何时hook进去都能被触发调用，在其中修改C#函数字节码，并想办法修改C#函数的编译状态为“尚未编译”，使得mono可以重新编译我们修改过的字节码。

经过一番调研，我选择了hook函数mono_jit_runtime_invoke。首先，要找到该函数在目标进程内的函数地址。我们可以通过逆向工程先找到函数在mono.dll文件中的相对地址，再加上之前获取到的基地址，即是将要Hook的目标函数地址。
```c
//该相对地址为逆向工程获得
#define JITRUNTIMEINVOKE 0xF137E

//声明一个函数变量
void* (*mono_jit_runtime_invoke)(void *method, void *obj, void **params, void **exc, void *error);

.........

//给该函数变量赋值
mono_jit_runtime_invoke = (void* (*)(void *method, void *obj, void **params, void **exc, void *error))((int)address + JITRUNTIMEINVOKE);
```
接着定义一个参数完全一致的hook函数my_mono_jit_runtime_invoke，通过easyhook工具来替换掉原来的mono_jit_runtime_invoke。
```c
HOOK_TRACE_INFO hHook = { NULL };

NTSTATUS result = LhInstallHook(
	mono_jit_runtime_invoke,
	my_mono_jit_runtime_invoke,
	NULL,
	&hHook);

if (FAILED(result))
{
	std::wstring s(RtlGetLastErrorString());
	std::wcout << "NativeInjectionEntryPoint: Failed to install hook: " << s << "\n";
	}
else
{
	std::cout << "NativeInjectionEntryPoint: Hook 'mono_jit_runtime_invoke installed successfully.\n";
}

ULONG ACLEntries[1] = { 0 };

LhSetExclusiveACL(ACLEntries, 1, &hHook);
void* my_mono_jit_runtime_invoke(void *method, void *obj, void **params, void **exc, void *error)
{
	char* name = mono_method_full_name(method, true);


	if (strstr(name, "FireOnPreRender") != 0)
		//if (strstr(name, "FixedUpdate") != 0)
	{
		if (status == H00K_BEGINE) {

			MonoMethodHeader* header = (MonoMethodHeader*)malloc(sizeof(MonoMethodHeader) + sizeof(MonoType*) * 1);
			memset(header, 0, sizeof(MonoMethodHeader) + sizeof(MonoType*) * 1);

			header->code = (const unsigned char  *)malloc(92);
			memcpy((void*)header->code, ILCODE, 92);
			header->code_size = 92;
			header->num_locals = 1;
			header->max_stack = 10;

			void* domain = mono_domain_get();
			//void* thread = mono_thread_attach(domain);
			void* assembly = mono_domain_assembly_open(domain, "D:/unityproject/UnityTest/test_Data/Managed/UnityEngine.dll");
			void* monoImage = mono_assembly_get_image(assembly);
			MonoType* type = mono_reflection_type_from_name("UnityEngine.GameObject", monoImage);

			*(MonoType**)((int)header + 0x18) = type;

			*(MonoMethodHeader**)((int)method + 0x18) = header;
 

			MonoMethodHeader* header2 = mono_method_get_header(method);

			std::cout << "my_mono_jit_runtime_invoke:" << name << "\n";

			status = H00K_FIND_METHOD;
		}
	}

	return mono_jit_runtime_invoke(method, obj, params, exc, error);
}
```
my_mono_jit_runtime_invoke函数做的事情主要就是修改上层C#函数的字节码数据。先根据name找到C#函数，然后将其中的code替换为了自己定义的字节码ILCODE，并相应修改了code_size，需要注意的是，还需要修改对应的MonoType，我这里是通过其内部函数mono_reflection_type_from_name动态获取得到。上面其实用到了很多mono的内部函数，都是通过逆向工程mono.dll得到的相对偏移再加上基地址而获得，对于固定的mono版本，那此方法是通用的。

当然，以上的ILCODE如何写，需要了解C#字节码的定义规范了，此处不赘述。

最后，还需要修改函数的状态为“未编译”。mono会将编译完的C#函数和结果用key-value的方式保存到一个hash table中，第二次调用时直接查询此table，如果存在就直接跳过编译步骤。所以我们必须将需要hook的函数编译结果从hash table中删除。这里我又hook进了另外一个函数mono_g_hash_table_lookup，因为这个函数的参数里可以获取到hash table和相应的key。
```c
HOOK_TRACE_INFO hHook2 = { NULL };

	NTSTATUS result2 = LhInstallHook(
		mono_g_hash_table_lookup,
		my_mono_g_hash_table_lookup,
		NULL,
		&hHook2);
int (*mono_g_hash_table_lookup)(void *hash_table,void* key);
int my_mono_g_hash_table_lookup(void *hash_table, void* key)
{

	int result = mono_g_hash_table_lookup(hash_table, key);
	if (status == H00K_FIND_METHOD)
	{
		char* name = mono_method_full_name(key, true);
		if (name != 0 && strstr(name, "FireOnPreRender") != 0)
		{
			std::cout << "my_mono_g_hash_table_lookup:" << name << "\n";
			if (result)
			{
				mono_g_hash_table_remove(hash_table, key);
			}
			void* domain = mono_domain_get();
			void* jit_hash = (void*)((int)domain + 0x80);

			if (mono_internal_hash_table_lookup(jit_hash, key))
			{
				mono_internal_hash_table_remove(jit_hash, key);
			}

			status = H00K_CHANGE_HASH_TABLE;
			return NULL;
		}
    }
	return result;
}
```
成功删除之前旧的编译结果后，mono就会自动编译并执行我们新插入的ILCODE了。

由于项目是针对我机器上的Mono版本，这里只给出了核心代码。

