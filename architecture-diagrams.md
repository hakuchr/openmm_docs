# 架构图集（Mermaid）

本篇汇总全系列文档用到的 Mermaid 图，便于集中浏览。GitHub、VSCode、Typora 等支持 Mermaid 的工具可直接渲染。

## 1. 三层架构总览

```mermaid
graph TB
    subgraph 应用层
        PY[Python openmm.app / openmm]
        CPP[C++ 用户代码]
        C_API[C API]
        FOR[Fortran API]
    end
    subgraph L1["第一层 公共API（openmmapi/）平台无关"]
        SYS[System]
        CTX[Context]
        FORCE[Force 家族 30+]
        INT[Integrator 家族 12+]
        STATE[State]
        LEM[LocalEnergyMinimizer]
    end
    subgraph L2["第二层 抽象分发层（olla/）"]
        PLAT[Platform 注册表]
        KF[KernelFactory]
        KERN[Kernel / KernelImpl]
        KERNS["kernels.h<br/>45 个抽象内核接口"]
    end
    subgraph L3["第三层 硬件后端（platforms/）"]
        REF[Reference 内置]
        CPU[CPU SIMD+多线程]
        COM[common 共享GPU层]
        CUDA[CUDA]
        OCL[OpenCL]
        HIP[HIP]
    end
    subgraph 插件["plugins/ 功能扩展"]
        AMO[amoeba 极化力场]
        DRU[drude 振子]
        RPMD[rpmd 环聚合物]
        CPME[cpupme CPU PME]
    end

    PY --> CPP
    C_API --> CPP
    FOR --> C_API
    CPP --> SYS
    CPP --> CTX
    CTX --> PLAT
    FORCE --> KERNS
    INT --> KERNS
    LEM --> CTX
    PLAT --> KF
    KF --> KERN
    KERN --> REF
    KERN --> CPU
    KERN --> COM
    COM --> CUDA
    COM --> OCL
    COM --> HIP
    PLAT -.插件注册.-> AMO
    PLAT -.插件注册.-> DRU
    PLAT -.插件注册.-> RPMD
    PLAT -.插件注册.-> CPME
```

## 2. 类关系图（核心 API）

```mermaid
classDiagram
    class System {
        +addParticle(mass)
        +addConstraint(p1,p2,d)
        +addForce(Force*)
        +setVirtualSite(i, VirtualSite*)
        +setDefaultPeriodicBoxVectors(a,b,c)
        +getNumParticles() int
        +getForce(i) Force
        +setForceGroup(i, g)
    }
    class Force {
        <<abstract>>
        +getKernelNames() vector~string~*
        +createImpl() ForceImpl*
        +updateParametersInContext(ctx)
        +setForceGroup(g)
    }
    class NonbondedForce
    class HarmonicBondForce
    class CustomNonbondedForce
    class Integrator {
        <<abstract>>
        +step(numSteps)
        +initialize(ContextImpl&)*
        +getKernelNames()*
        +setIntegrationForceGroups(mask)
        +computeKineticEnergy() double
    }
    class LangevinMiddleIntegrator
    class VerletIntegrator
    class CustomIntegrator
    class Context {
        +Context(System, Integrator[, Platform[, props]])
        +setPositions(vec)
        +setVelocitiesToTemperature(T, seed)
        +getState(types, enforcePBC, groups) State
        +getPlatform() Platform
        +applyConstraints(tol)
        +reinitialize(preserveState)
        +createCheckpoint(os) / loadCheckpoint(is)
        +setParameter(name, val)
    }
    class ContextImpl {
        -vector~ForceImpl~ forceImpls
        -Kernel initializeForcesKernel
        -Kernel updateStateDataKernel
        -Kernel applyConstraintsKernel
        -Kernel virtualSitesKernel
        -Kernel minimizeKernel
        -void* platformData
        +calcForcesAndEnergy(includeF, includeE, groups)
        +updateContextState()
        +applyConstraints(tol)
        +computeVirtualSites()
        +calcKineticEnergy()
        +minimize(...)
    }
    class State {
        +getPositions() vector~Vec3~
        +getVelocities() vector~Vec3~
        +getForces() vector~Vec3~
        +getPotentialEnergy() double
        +getKineticEnergy() double
        +getTime() double
        +getPeriodicBoxVectors()
    }
    class Platform {
        <<abstract>>
        +getName() string*
        +getSpeed() double*
        +supportsDoublePrecision() bool*
        +registerKernelFactory(name, factory)
        +createKernel(name, ctx) Kernel
        +contextCreated(ctx, props)
        +registerPlatform(p) static
        +findPlatform(names) static Platform
        +loadPluginsFromDirectory(dir) static
    }
    class Kernel {
        +getAs~T~() T
        +isEmpty() bool
    }
    class KernelImpl {
        <<abstract>>
        #const Platform* platform
    }
    class KernelFactory {
        <<abstract>>
        +createKernelImpl(name, plat, ctx) KernelImpl*
    }

    Force <|-- NonbondedForce
    Force <|-- HarmonicBondForce
    Force <|-- CustomNonbondedForce
    Integrator <|-- LangevinMiddleIntegrator
    Integrator <|-- VerletIntegrator
    Integrator <|-- CustomIntegrator
    System o-- Force : 持有多个
    Context --> System : 引用
    Context --> Integrator : 引用
    Context ..> State : getState
    Context *-- ContextImpl : PIMPL
    ContextImpl --> Platform : 持有
    Platform --> KernelFactory : 注册
    Platform ..> Kernel : createKernel
    Kernel *-- KernelImpl
    KernelImpl <|-- CalcNonbondedForceKernel
    ContextImpl ..> Kernel : 缓存
```

## 3. Platform / Kernel 分发时序

```mermaid
sequenceDiagram
    participant User
    participant Ctx as Context
    participant CI as ContextImpl
    participant Plat as Platform
    participant KF as KernelFactory
    participant KImpl as CudaCalcNonbondedForceKernel

    User->>Ctx: Context(system, integ)
    Ctx->>CI: new ContextImpl(...)
    CI->>CI: 收集 kernelNames (ForceImpl + Integrator)
    CI->>Plat: findPlatform(kernelNames)
    Plat-->>CI: CudaPlatform (speed 最高)
    CI->>Plat: contextCreated(this, props)
    Plat->>Plat: 创建 CudaContext (设备/流)
    loop 每个 Force
        CI->>CI: force.createImpl() → ForceImpl
        CI->>CI: ForceImpl.initialize(*this)
        CI->>Plat: createKernel("CalcNonbondedForceKernel", ctx)
        Plat->>KF: createKernelImpl(name, ...)
        KF-->>Plat: new CudaCalcNonbondedForceKernel(ctx)
        Plat-->>CI: Kernel
        CI->>KImpl: kernel.getAs<...>().initialize(system)
    end
    CI->>CI: 缓存共用内核 (updateStateData/...)
    CI-->>Ctx: 构造完成
    User->>Ctx: integ.step(1)
    Ctx->>CI: step
    CI->>CI: updateContextState (恒温/恒压)
    CI->>CI: calcForcesAndEnergy
    CI->>KImpl: kernel.getAs<...>().execute(...)
    KImpl->>KImpl: 启动 CUDA 内核
    KImpl-->>CI: 力/能量
    CI->>CI: IntegrateLangevinMiddleStepKernel.execute
    CI->>CI: applyConstraints + virtualSites
    CI-->>Ctx: step 返回
    User->>Ctx: getState(Positions|Energy)
    Ctx->>CI: 构建 State
    CI-->>Ctx: State
    Ctx-->>User: State
```

## 4. 平台选择算法

```mermaid
flowchart TD
    A[Context 构造] --> B{用户传入 Platform?}
    B -- 是 --> C[校验 supportsKernels]
    C --> C1{支持?}
    C1 -- 否 --> E[抛异常]
    C1 -- 是 --> F[使用该平台]
    B -- 否 --> G{OPENMM_DEFAULT_PLATFORM<br/>环境变量?}
    G -- 是 --> H[getPlatformByName]
    H --> H1{支持内核?}
    H1 -- 否 --> E
    H1 -- 是 --> F
    G -- 否 --> I[遍历所有已注册平台]
    I --> J[收集 supportsKernels 为真的平台]
    J --> K[按 getSpeed 降序排序]
    K --> L{尝试最快平台<br/>contextCreated}
    L -- 成功 --> F
    L -- 失败/异常 --> M{还有候选?}
    M -- 是 --> L
    M -- 否 --> E
```

## 5. 插件两阶段加载

```mermaid
sequenceDiagram
    participant App as 应用进程
    participant Lib as libOpenMM
    participant Loader as Platform::loadPluginsFromDirectory
    participant P1 as 插件1 .so
    participant P2 as 插件2 .so
    participant Reg as Platform 注册表

    App->>Lib: 链接启动
    Lib->>Reg: 静态注册 ReferencePlatform<br/>(olla/src/Platform.cpp:59)
    App->>Loader: loadPluginsFromDirectory(pluginsDir)
    Loader->>P1: dlopen
    Loader->>P2: dlopen
    Note over Loader: 第一阶段：全部 registerPlatforms()
    Loader->>P1: registerPlatforms()
    P1->>Reg: Platform::registerPlatform(new CpuPlatform())
    Loader->>P2: registerPlatforms()
    P2->>Reg: Platform::registerPlatform(new CudaPlatform())
    Note over Loader: 第二阶段：全部 registerKernelFactories()
    Loader->>P1: registerKernelFactories()
    P1->>Reg: cudaPlatform.registerKernelFactory(name, factory)
    Loader->>P2: registerKernelFactories()
    P2->>Reg: cudaPlatform.registerKernelFactory(Amoeba内核, factory)
    App->>App: Context 创建 → findPlatform 自动选最快
```

## 6. common 共享 GPU 层架构

```mermaid
graph LR
    subgraph common["platforms/common/ 共享层"]
        A1["ComputeContext 抽象<br/>ComputeKernel / ComputeProgram<br/>ComputeArray / ComputeEvent / ComputeQueue"]
        A2["CommonXxxKernel 主机内核<br/>~40 个，编排设备内核"]
        A3["kernels/*.cc 共享设备源<br/>66 个，平台无关方言"]
    end
    subgraph CUDA["platforms/cuda/"]
        CC[CudaContext : ComputeContext]
        CK[CudaKernel/Array/Program]
        CS["kernels/*.cu (9)<br/>common.cu 宏层 + 原语"]
    end
    subgraph OCL["platforms/opencl/"]
        OC[OpenCLContext : ComputeContext]
        OK[OpenCLKernel/Array/Program]
        OS["kernels/*.cl (10)<br/>common.cl 宏层"]
    end
    subgraph HIP["platforms/hip/"]
        HC[HipContext : ComputeContext]
        HK[HipKernel/Array/Program]
        HS["kernels/*.hip (8)<br/>common.hip 宏层"]
    end
    A1 -.继承.-> CC
    A1 -.继承.-> OC
    A1 -.继承.-> HC
    A2 -.实例化.-> CC
    A2 -.实例化.-> OC
    A2 -.实例化.-> HC
    A3 -.编译.-> CS
    A3 -.编译.-> OS
    A3 -.编译.-> HS
```

## 7. 设备数据流（GPU/NPU 视角）

```mermaid
flowchart LR
    subgraph 主机
        H1[用户 setPositions<br/>vector Vec3]
        H2[State 输出<br/>vector Vec3]
    end
    subgraph 设备
        D1[posq 缓冲<br/>float4 数组]
        D2[velm 缓冲]
        D3[forceAcc 缓冲]
        D4[内核执行<br/>nonbonded.cc / verlet.cc / ...]
    end
    H1 -->|UpdateStateDataKernel<br/>setPositions| D1
    D4 -->|读 posq| D1
    D4 -->|读 velm| D2
    D4 -->|写 forceAcc| D3
    D4 -->|更新 posq/velm| D1
    D1 -->|getState 拷回| H2
```

## 8. 模拟循环内部

```mermaid
flowchart TD
    S[Integrator::step n] --> S1[循环 n 次]
    S1 --> U[ContextImpl::updateContextState]
    U --> U1[各 ForceImpl::updateContextState<br/>AndersenThermostat / MonteCarloBarostat /<br/>CMMotionRemover 修改速度/盒子]
    U1 --> C[ContextImpl::calcForcesAndEnergy<br/>includeForces, includeEnergy, groups]
    C --> C1[CalcForcesAndEnergyKernel::beginComputation]
    C1 --> C2{每个力组匹配的 ForceImpl}
    C2 --> C3[ForceImpl::calcForcesAndEnergy]
    C3 --> C4[kernel.getAs CalcXxxForceKernel .execute<br/>启动设备内核 / CPU SIMD]
    C4 --> C5[累加力/能量]
    C5 --> C6[CalcForcesAndEnergyKernel::endComputation<br/>返回势能]
    C6 --> I[IntegrateXxxStepKernel::execute<br/>更新位置/速度]
    I --> AC[ContextImpl::applyConstraints<br/>ApplyConstraintsKernel SETTLE/SHAKE]
    AC --> VS[ContextImpl::computeVirtualSites<br/>VirtualSitesKernel]
    VS --> S1
```

## 9. 语言封装生成流

```mermaid
flowchart LR
    H[C++ 公共头<br/>openmmapi/include/openmm/*.h] -->|doxygen| DX[Doxygen XML]
    DX --> GW[generateWrappers.py<br/>自研生成器]
    GW --> CH[OpenMMCWrapper.h]
    GW --> CS[OpenMMCWrapper.cpp]
    GW --> FH[OpenMMFortranModule.f90]
    GW --> FS[OpenMMFortranWrapper.cpp]
    CS -.编译进.-> LIB[libOpenMM]
    FS -.编译进.-> LIB
    DX --> SW[swigInputBuilder.py]
    SW --> SI[OpenMMSwigHeaders.i<br/>+ docstring.i]
    SI --> M[OpenMM.i 主接口]
    M -->|swig -python -c++| CXX[OpenMMSwig.cxx]
    M -->|swig| PY[openmm.py]
    CXX -->|setup.py build| EXT[_openmm 扩展]
    EXT -.链接.-> LIB
```

## 10. NPU 平台 MVP 验证路径

```mermaid
flowchart LR
    S1[1. 平台注册<br/>getPlatformByName NPU] --> S2[2. Context 创建<br/>NpuContext 初始化设备]
    S2 --> S3[3. UpdateStateData<br/>setPositions/getState 拷贝]
    S3 --> S4[4. HarmonicBondForce<br/>最简单力]
    S4 --> S5[5. VerletIntegrator<br/>NVE 一步]
    S5 --> S6[6. NonbondedForce 无 cutoff<br/>真空非键]
    S6 --> S7[7. PME 周期边界<br/>NpuFFT3D]
    S7 --> S8[8. LangevinMiddleIntegrator<br/>恒温 RNG]
    S8 --> S9[9. CustomNonbondedForce<br/>Lepton 表达式转译]
    S9 --> S10[10. 约束 SETTLE<br/>水模型]
    S10 --> S11[11. 完整蛋白-配体 MD]
```

## 11. NPU 适配工作量分布

```mermaid
pie title NPU 适配工作量分布（估算）
    "NpuContext 设备上下文" : 25
    "设备内核源 .npu（66共享 + 8原语）" : 30
    "NpuCalcNonbondedForceKernel 非键优化" : 20
    "NpuFFT3D PME" : 10
    "common.npu 宏层 + intrinsics" : 8
    "工具子系统 / CMake / 测试" : 7
```

## 12. 第三方库依赖关系

```mermaid
graph LR
    subgraph libraries/
        L[lepton 表达式]
        A[asmjit JIT]
        V[vecmath SIMD]
        S[sfmt 随机数]
        B[lbfgs 优化]
        P[pocketfft CPU FFT]
        K[vkfft GPU FFT]
        H[hilbert 曲线]
        C[csha1 哈希]
        I[irrxml XML]
        J[jama/quern 线代]
    end
    L --> CF[Custom Force/Integrator]
    A --> L
    V --> CPU[CPU 平台]
    S --> RND[随机积分器/恒温器]
    B --> MIN[能量最小化]
    P --> RPME[Reference PME]
    K --> GPME[GPU/NPU PME]
    H --> NB[邻居表重排]
    C --> KC[内核缓存]
    I --> SER[序列化 XML]
    J --> REF[Reference 力计算]
```

---

更多细节见各专题文档。回到 [README.md](README.md)。
