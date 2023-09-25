# Reusable Enclaves for Confidential Serverless Computing

## About the project

This is a research project aims to solve the cold start problem without sacrificing the security by creating a method to securely reuse the enclave. The paper is accepted to 2023 USENIX Security Symposium. You can download the paper [here](https://u.osu.edu/zhao-3289/files/2023/07/Reusable-Enclave.pdf).

This repository is a guide to build and use each component of this project. The project involves 6 other projects:
+ Apache OpenWhisk
+ Intel SGX SDK
+ WOW
+ LLVM
+ WAMR
+ DCAP end-to-end encryption infrastructure

All implementations except for existing code bases (Intel SGX SDK, OpenWhisk, WOW, WAMR, etc.) were written and debugged by NSKernel.

## License

This project itself is opensourced under GPLv2. See LICENSE.

Copyright (C) 2022 NSKernel and OSU SecLab.

+ Apache OpenWhisk is licnesed under Apache 2.0 License.
+ Intel SGX SDK is licensed under GPLv2.
+ WOW did not indicate its license.
+ LLVM is licensed under Apache 2.0 License with LLVM exceptions.
+ WAMR is licensed under Apache 2.0 License.

## How to build and use

This project is very complicated and I strongly encourage you to have background knowledge of each of the components before you try to build and use the project.

Disclaimer: THERE IS NO WARRANTY TO THE SOFTWARE. THE SOFTWARE IS NOT BUG FREE AND MAY CAUSE DAMANGE TO YOUR SYSTEM. UNDER NO CIRCUMSTANCES SHALL THE AUTHORS OF THE SOFTWARE BE RESPONSIBILE TO ANY DAMANGE OR LOSS DUE TO THE USE OF THE SOFTWARE.

### Step 1. Build the LLVM

Clone the modified LLVM

```
git clone https://github.com/NSKernel/llvm-reusable-enclaves.git
```

Build the LLVM

```
cd llvm-reusable-enclaves
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm
make
```

### Step 2. Build the instrumented Intel SGX SDK

Download and build a regular Intel SGX SDK's PSW part
```
https://github.com/intel/linux-sgx.git
cd linux-sgx
make preparation
make deb_psw_pkg
```
Install the built debs.

Clone the modified Intel SGX SDK
```
git clone https://github.com/NSKernel/linux-sgx-san.git
```

Prepare the building
```
cd linux-sgx-san
# Follow the SDK's instruction to prepare the build
```

Manually build the IPP crypto library using our toolchain
```
cd external/ippcp_internal
```
Modify the `Makefile` to use our toolchain modifying line 81 into
```
cd $(IPP_SOURCE) && CC='your/llvm/build/bin/clang' CXX='your/llvm/build/bin/clang++' $(PRE_CONFIG) cmake CMakeLists.txt $(IPP_CONFIG) && cd build && make ippcp_s -j14
```
Build IPP crypto
```
make ipp_source
make
```

Copy `sgx_ippcp.h` to the inc folder
```
cp ~/linux-sgx/external/ippcp_internal/inc/sgx_ippcp.h ~/linux-sgx-san/external/ippcp_internal/inc/
```

Build the SDK
```
cd linux-sgx-san
CC='your/llvm/build/bin/clang' CXX='your/llvm/build/bin/clang++' make sdk_install_pkg
```

Install the SDK and setup the environment
```
cd linux/installer/bin/
./sgx_linux_x64_sdk*
source sgxsdk/environment
```

Note that you will have to clone a clean Intel SGX SDK and build its PSW using regular GCC. You will need to install that PSW.

### Step 3. Build the WAMR and WOW

Clone the WOW project
```
git clone https://github.com/NSKernel/wow.git
```

Clone the WAMR into `wamr-sys`
```
cd wow/wamr-sys
git clone https://github.com/NSKernel/wasm-micro-runtime.git
cd ..
```

Build WOW and WAMR
```
cargo build --manifest-path ./ow-executor-san/Cargo.toml --release --features wamr_rt
```

Copy the built enclave to the executor path
```
cp wamr-sys/wasm-micro-runtime/core/shared/platform/linux-sgx/Enclave-san/enclave.signed.so target/release/
```

### Step 5. Build a serverless app

Following the WOW's README,
```
cargo build --manifest-path ./ow-wasm-action/Cargo.toml --release --example add --target wasm32-wasi --no-default-features --features wasm
```

### Step 6. Build the DCAP end-to-end infrastructure

Clone the project
```
git clone https://github.com/NSKernel/reusable-enclaves-dcap.git
```

Build the project
```
cd reusable-enclaves-dcap
make
```

### Step 7. Build and run OpenWhisk

Clone the OpenWhisk
```
git clone https://github.com/NSKernel/openwhisk-reusable-enclaves.git
```

Launch OpenWhisk
```
./gradlew core:standalone:bootRun
```

### Step 8. Setup environemnt

New terminal 1:
```
cd reusable-enclaves-dcap/bin
./gatewayapp
```

New terminal 2:
```
cd reusable-enclaves-dcap/bin
./service_provider
```
In the prompt, enter the compiled WASM file to encrypt (e.g., `target/wasm32-wasi/release/examples/add.wasm`)

New terminal 3:
```
cd wow
./target/release/executor-san
```

New terminal 4: Zip the encrypted WASM file
```
zip add.zip target/wasm32-wasi/release/examples/add.wasm.enc
```
Use WSK to execute the zip file following WOW's README.

### (Optional) Step 9. Build the AoT infrastructure

cd wow/wamr-sys/wasm-micro-runtime/wamr-compiler
```
mkdir build
```

Edit the CMakeLists.txt's Line 115's LLVM path into your LLVM. Then build the WAMR compiler

```
cd build
cmake ../
make
```

### (Optional) Step 10. Compile AoT code

```
cd wow/wamr-sys/wasm-micro-runtime/wamr-compiler/build/
./wamrc target/wasm32-wasi/release/examples/add.wasm  -o target/wasm32-wasi/release/examples/add.aot
```
Go back to Step 8 and follow it except for changing `add.wasm` into `add.aot`.

