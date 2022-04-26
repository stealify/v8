# Directly Load and Interact with wasm via Isolates


WasmCompileModule === v8::WasmModuleObject  in newer versions

```cpp
Local<WasmCompiledModule> module = WasmCompiledModule::DeserializeOrCompile(isolate,
            WasmCompiledModule::BufferReference(0, 0),
            WasmCompiledModule::BufferReference(wasmbin.data(), wasmbin.size())
        ).ToLocalChecked();
```

v8 doesn't expose its webassembly api directly, you have to get them from JS global context. The following code creates a module instance, and gets the exports of the instance:

```cpp
    using args_type = Local<Value>[];

    Local<Object> module_instance_exports = context->Global()
        ->Get(context, String::NewFromUtf8(isolate, "WebAssembly"))
        .ToLocalChecked().As<Object>()
        ->Get(context, String::NewFromUtf8(isolate, "Instance"))
        .ToLocalChecked().As<Object>()
        ->CallAsConstructor(context, 1, args_type{module})
        .ToLocalChecked().As<Object>()
        ->Get(context, String::NewFromUtf8(isolate, "exports"))
        .ToLocalChecked().As<Object>()
        ;
```
Then you can get the add function from exports object and call it:
```cpp
    Local<Int32> adder_res = module_instance_exports
        ->Get(context, String::NewFromUtf8(isolate, "add"))
        .ToLocalChecked().As<Function>()
        ->Call(context, context->Global(), 2, args_type{Int32::New(isolate, 77), Int32::New(isolate, 88)})
        .ToLocalChecked().As<Int32>();

    std::cout << "77 + 88 = " << adder_res->Value() << "\n";
 ```
 
 
 ## The Final Result with example wasm Code 
 ```cpp
 #include <include/v8.h>

#include <include/libplatform/libplatform.h>

using v8::HandleScope;
using v8::Isolate;
using v8::Local;
using v8::Promise;
using v8::WasmModuleObjectBuilderStreaming;
using v8::WasmCompiledModule;
using v8::Context;
using v8::Local;
using v8::Value;
using v8::String;
using v8::Object;
using v8::Function;
using v8::Int32;
using args_type = Local<Value>[];

int main(int argc, char* argv[]) {
  v8::V8::InitializeICUDefaultLocation(argv[0]);
  v8::V8::InitializeExternalStartupData(argv[0]);
  std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();
  Isolate::CreateParams create_params;
  create_params.array_buffer_allocator = v8::ArrayBuffer::Allocator::NewDefaultAllocator();
  Isolate* isolate = Isolate::New(create_params);
  Isolate::Scope isolate_scope(isolate);
  HandleScope scope(isolate);
  Local<Context> context = Context::New(isolate);
  Context::Scope context_scope(context);

  WasmModuleObjectBuilderStreaming stream(isolate);

  // Use the v8 API to generate a WebAssembly module.
  //
  // |bytes| contains the binary format for the following module: //
  //     (func (export "add") (param i32 i32) (result i32)
  //       get_local 0
  //       get_local 1
  //       i32.add)
  //
  // taken from: https://github.com/v8/v8/blob/master/samples/hello-world.cc#L66
  std::vector<uint8_t> wasmbin {
          0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00, 0x01, 0x07, 0x01,
          0x60, 0x02, 0x7f, 0x7f, 0x01, 0x7f, 0x03, 0x02, 0x01, 0x00, 0x07,
          0x07, 0x01, 0x03, 0x61, 0x64, 0x64, 0x00, 0x00, 0x0a, 0x09, 0x01,
          0x07, 0x00, 0x20, 0x00, 0x20, 0x01, 0x6a, 0x0b
  };

  // same as calling:
  // let module = new WebAssembly.Module(bytes);
  Local<WasmCompiledModule> module = WasmCompiledModule::DeserializeOrCompile(isolate,
      WasmCompiledModule::BufferReference(0, 0),
      WasmCompiledModule::BufferReference(wasmbin.data(), wasmbin.size())
      ).ToLocalChecked();

  // same as calling:
  // let module_instance_exports = new WebAssembly.Instance(module).exports;
  args_type instance_args{module};
  Local<Object> module_instance_exports = context->Global()
    ->Get(context, String::NewFromUtf8(isolate, "WebAssembly"))
    .ToLocalChecked().As<Object>()
    ->Get(context, String::NewFromUtf8(isolate, "Instance"))
    .ToLocalChecked().As<Object>()
    ->CallAsConstructor(context, 1, instance_args)
    .ToLocalChecked().As<Object>()
    ->Get(context, String::NewFromUtf8(isolate, "exports"))
    .ToLocalChecked().As<Object>()
    ;

  // same as calling:
  // module_instance_exports.add(77, 88)
  args_type add_args{Int32::New(isolate, 77), Int32::New(isolate, 88)};
  Local<Int32> adder_res = module_instance_exports
    ->Get(context, String::NewFromUtf8(isolate, "add"))
    .ToLocalChecked().As<Function>()
    ->Call(context, context->Global(), 2, add_args)
    .ToLocalChecked().As<Int32>();

  printf("77 + 88 = %d\n", adder_res->Value());
  return 0;
}
 ```
