# API参考

<cite>
**本文档中引用的文件**
- [mod.rs](file://src/vmm/mod.rs)
- [config.rs](file://src/vmm/config.rs)
- [vcpus.rs](file://src/vmm/vcpus.rs)
- [hvc.rs](file://src/vmm/hvc.rs)
- [ivc.rs](file://src/vmm/ivc.rs)
- [api.rs](file://src/hal/arch/aarch64/api.rs)
</cite>

## 目录
1. [VMM模块API](#vmm模块api)
2. [VM配置结构体](#vm配置结构体)
3. [vCPU管理API](#vcpu管理api)
4. [超调用处理机制](#超调用处理机制)
5. [IVC通道通信接口](#ivc通道通信接口)
6. [HAL层底层操作原语](#hal层底层操作原语)

## VMM模块API

### `vmm::init()`
初始化虚拟机监控器（VMM）。

**功能描述**  
该函数负责初始化VMM，包括根据配置文件创建虚拟机结构，并为每个虚拟机设置主vCPU任务。

**调用时机**  
应在系统启动早期、多任务调度开始前调用此函数。

**参数说明**  
无显式参数。

**返回值**  
无返回值。

**可能的错误码**  
不直接返回错误码，但内部会记录日志信息。

**使用示例**  
```rust
vmm::init();
```

**Section sources**
- [mod.rs](file://src/vmm/mod.rs#L30-L47)

### `vmm::start()`
启动VMM并引导所有虚拟机。

**功能描述**  
遍历所有已注册的虚拟机，尝试启动它们。对于成功启动的虚拟机，增加运行计数；失败则记录警告。随后进入等待状态，直到所有虚拟机停止运行。

**调用时机**  
在 `vmm::init()` 成功执行后调用。

**参数说明**  
无显式参数。

**返回值**  
无返回值。

**可能的错误码**  
启动失败时会在日志中输出具体错误信息。

**使用示例**  
```rust
vmm::start();
```

**Section sources**
- [mod.rs](file://src/vmm/mod.rs#L49-L78)

### `with_vm<T>`
在指定虚拟机上下文中执行闭包。

**功能描述**  
提供一种安全的方式来访问特定ID的虚拟机实例，并在其上执行操作。

**参数说明**  
- `vm_id`: 虚拟机唯一标识符。
- `f`: 一个接受 `VMRef` 并返回类型 `T` 的闭包。

**返回值**  
若找到对应虚拟机，则返回 `Some(T)`；否则返回 `None`。

**使用示例**  
```rust
vmm::with_vm(0, |vm| {
    info!("当前VM名称: {}", vm.name());
});
```

**Section sources**
- [mod.rs](file://src/vmm/mod.rs#L80-L88)

### `with_vm_and_vcpu<T>`
在指定虚拟机和vCPU上下文中执行闭包。

**功能描述**  
类似于 `with_vm`，但同时要求获取特定vCPU的引用。

**参数说明**  
- `vm_id`: 虚拟机ID。
- `vcpu_id`: vCPU ID。
- `f`: 接受 `VMRef` 和 `VCpuRef` 的闭包。

**返回值**  
成功时返回 `Some(T)`，否则 `None`。

**使用示例**  
```rust
vmm::with_vm_and_vcpu(0, 0, |vm, vcpu| {
    info!("VCPU状态: {:?}", vcpu.state());
});
```

**Section sources**
- [mod.rs](file://src/vmm/mod.rs#L90-L101)

## VM配置结构体

位于 `vmm::config` 模块中的 `AxVMConfig` 结构体用于定义虚拟机的资源配置。

### 主要字段及其约束条件

#### 内存区域 (`memory_regions`)
- **类型**: `Vec<VMMemoryRegion>`
- **约束**: 每个内存区域必须具有唯一的物理地址范围，且大小需对齐到2MB。
- **用途**: 定义客户操作系统可用的物理内存布局。

#### 设备透传配置 (`pass_through_devices`)
- **类型**: `Vec<PassThroughDeviceConfig>`
- **约束**: GPA与HPA地址不能重叠现有内存区域。
- **用途**: 允许虚拟机直接访问主机硬件设备。

#### CPU配置 (`cpu_config`)
- **字段**:
  - `bsp_entry`: 引导处理器入口地址。
  - `ap_entry`: 应用处理器入口地址。
- **约束**: 必须指向有效的可执行代码段。

#### 镜像配置 (`image_config`)
- **字段**:
  - `kernel_load_gpa`: 内核加载的客户物理地址。
  - `dtb_load_gpa`: 设备树二进制文件加载地址。
- **约束**: 地址应避开BIOS保留区域（通常前2MB）。

**Section sources**
- [config.rs](file://src/vmm/config.rs#L10-L284)

## vCPU管理API

### `setup_vm_primary_vcpu(vm)`
为指定虚拟机设置主vCPU。

**功能描述**  
初始化虚拟机的第一个vCPU（通常是BSP），并为其分配内核任务。

**参数说明**  
- `vm`: `VMRef` 类型，表示目标虚拟机。

**返回值**  
无返回值。

**调用时机**  
在 `vmm::init()` 中被调用。

**Section sources**
- [vcpus.rs](file://src/vmm/vcpus.rs#L358-L388)

### `notify_primary_vcpu(vm_id)`
通知主vCPU可以开始运行。

**功能描述**  
唤醒处于等待状态的主vCPU任务，使其进入执行循环。

**参数说明**  
- `vm_id`: 目标虚拟机ID。

**返回值**  
无返回值。

**调用时机**  
在 `vmm::start()` 启动虚拟机后调用。

**Section sources**
- [vcpus.rs](file://src/vmm/vcpus.rs#L346-L356)

### `alloc_vcpu_task(vm, vcpu)`
为vCPU分配ArceOS任务。

**功能描述**  
创建一个新的内核任务来承载vCPU的执行流，设置其入口函数为 `vcpu_run`，并初始化扩展上下文。

**参数说明**  
- `vm`: 所属虚拟机引用。
- `vcpu`: vCPU引用。

**返回值**  
返回新创建的任务引用 `AxTaskRef`。

**使用示例**  
由内部机制自动调用，一般无需用户手动触发。

**Section sources**
- [vcpus.rs](file://src/vmm/vcpus.rs#L390-L424)

### `vcpu_run()`
vCPU执行主循环。

**功能描述**  
这是每个vCPU任务的入口点。它会持续运行虚拟CPU，处理各种退出原因，如超调用、中断等。

**调用时机**  
由任务调度器在vCPU任务被唤醒时自动调用。

**错误处理**  
遇到不可恢复错误时将关闭整个虚拟机。

**Section sources**
- [vcpus.rs](file://src/vmm/vcpus.rs#L426-L506)

## 超调用处理机制

### `HyperCall::new(vcpu, vm, code, args)`
构造一个新的超调用对象。

**功能描述**  
解析来自客户机的超调用请求，验证调用号合法性。

**参数说明**  
- `vcpu`: 发起调用的vCPU。
- `vm`: 所属虚拟机。
- `code`: 超调用编号。
- `args`: 参数数组。

**返回值**  
成功返回 `Ok(HyperCall)`，无效调用号返回 `Err(AxError::InvalidInput)`。

**Section sources**
- [hvc.rs](file://src/vmm/hvc.rs#L15-L30)

### `HyperCall::execute()`
执行超调用。

**支持的操作码**:
- `HIVCPublishChannel`: 发布IVC共享通道。
- `HIVCUnPublishChannel`: 撤销发布通道。
- `HIVCSubscribChannel`: 订阅远程IVC通道。
- `HIVCUnSubscribChannel`: 取消订阅。

**参数传递方式**  
通过GPR寄存器传递最多6个参数。

**返回值**  
遵循标准系统调用约定，成功返回0，失败返回负数错误码。

**错误码**:
- `-1`: 执行失败。
- `Unsupported`: 不支持的调用号。

**Section sources**
- [hvc.rs](file://src/vmm/hvc.rs#L32-L146)

## IVC通道通信接口

### `insert_channel(publisher_vm_id, channel)`
插入一个新的IVC通道。

**功能描述**  
将由某个虚拟机发布的通道注册到全局表中。

**参数说明**  
- `publisher_vm_id`: 发布者虚拟机ID。
- `channel`: 通道实例。

**返回值**  
存在重复键时返回 `AlreadyExists` 错误。

**Section sources**
- [ivc.rs](file://src/vmm/ivc.rs#L15-L28)

### `unpublish_channel(publisher_vm_id, key)`
撤销发布指定通道。

**功能描述**  
标记通道为未发布状态。若有订阅者仍存在，则保留元数据；否则彻底删除。

**返回值**  
返回 `(base_gpa, size)` 或 `NotFound` 错误。

**Section sources**
- [ivc.rs](file://src/vmm/ivc.rs#L30-L62)

### `subscribe_to_channel_of_publisher(...)`
订阅其他虚拟机发布的通道。

**功能描述**  
建立跨虚拟机共享内存连接。

**参数说明**  
- `publisher_vm_id`, `key`: 定位源通道。
- `subscriber_vm_id`, `subscriber_gpa`: 提供本地映射信息。

**返回值**  
成功返回 `(HostPhysAddr, usize)`，失败返回 `NotFound`。

**Section sources**
- [ivc.rs](file://src/vmm/ivc.rs#L98-L122)

### `IVCChannel::alloc(...)`
分配新的IVC通道资源。

**功能描述**  
在物理内存中分配一页作为共享区域，并初始化头部信息。

**约束条件**  
最大共享区域限制为4KB。

**返回值**  
成功返回 `Ok(IVCChannel)`，内存不足返回 `NoMemory`。

**Section sources**
- [ivc.rs](file://src/vmm/ivc.rs#L208-L244)

## HAL层底层操作原语

位于 `hal::arch::aarch64::api.rs` 的外部函数接口。

### `hardware_inject_virtual_interrupt(irq)`
注入虚拟中断。

**功能描述**  
向当前vCPU注入指定中断向量。

**参数说明**  
- `irq`: 中断向量号。

**调用路径**  
由VMM内部中断模拟逻辑调用。

**Section sources**
- [api.rs](file://src/hal/arch/aarch64/api.rs#L5-L9)

### `read_vgicd_typer()`
读取虚拟GIC Distributor TYPER寄存器。

**功能描述**  
返回虚拟中断控制器的能力信息。

**返回值**  
32位寄存器原始值。

**适用架构**  
ARM GICv2/v3。

**Section sources**
- [api.rs](file://src/hal/arch/aarch64/api.rs#L11-L24)

### `get_host_gicd_base()`
获取主机GICD基地址。

**功能描述**  
通过驱动查找机制定位物理GIC Distributor寄存器块的物理地址。

**返回值**  
`PhysAddr` 类型的基地址。

**异常行为**  
找不到驱动时会触发panic。

**Section sources**
- [api.rs](file://src/hal/arch/aarch64/api.rs#L49-L62)

### `get_host_gicr_base()`
获取主机GICR基地址。

**功能描述**  
仅适用于GICv3架构，返回Redistributor寄存器区域的物理地址。

**返回值**  
`PhysAddr` 类型的基地址。

**异常行为**  
仅支持GICv3，否则panic。

**Section sources**
- [api.rs](file://src/hal/arch/aarch64/api.rs#L64-L74)