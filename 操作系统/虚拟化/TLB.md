## 问题
使用分页作为核心机制来实现虚拟内存，可能会带来较高的性能开销。因为要使用分页，就要将内存地址空间切分成大量固定大小的单元（页）​，并且需要记录这些单元的地址映射信息。因为这些映射信息一般存储在物理内存中，所以在转换虚拟地址时，==分页逻辑上需要一次额外的内存访问。==每次指令获取、显式加载或保存，都要额外读一次内存以得到转换信息，这慢得无法接受。

为了找到一个虚拟地址对应的实际物理内存地址，处理器（或内存管理单元 MMU）需要先去查询一个存储在内存中的“地图”（页表），这个查询本身就是一次内存访问，然后才能根据查到的信息去访问真正想要的数据所在的物理内存地址。

**核心概念：**

1. **虚拟地址 (Virtual Address):** 程序运行时使用的地址。每个程序都认为自己拥有一块连续的、私有的内存空间。
2. **物理地址 (Physical Address):** 内存条（RAM）上真实的、硬件能够识别的地址。
3. **页表 (Page Table):** 操作系统为每个进程维护的一个数据结构，它记录了虚拟地址中的“页”如何映射到物理内存中的“页帧”。可以把它想象成一个翻译词典，左边是虚拟页号，右边是对应的物理页帧号。
4. **内存管理单元 (MMU):** CPU 内部的一个硬件单元，负责将程序使用的虚拟地址实时翻译成物理地址。

**为什么需要“额外的内存访问”？**
想象一下没有分页的简单情况：
- CPU 想读取地址 `1000` 处的数据。
- CPU 直接把地址 `1000` 发送到内存总线上。
- 内存控制器从物理内存的 `1000` 位置取出数据给 CPU。
- **总共只需要一次内存访问**（访问地址 `1000` 处的数据）。

现在，在有分页的情况下：
- CPU 想读取**虚拟地址** `V_Addr = 1000` 处的数据。
- MMU 介入，需要将 `V_Addr` 翻译成**物理地址** `P_Addr`。
- 为了翻译，MMU 需要查找**页表**。这个页表存储在哪里？—— **存储在物理内存中**。
- **第一次内存访问（额外的访问）：** MMU 计算出包含 `V_Addr` 映射信息的**页表项 (Page Table Entry, PTE)** 在物理内存中的地址 (比如 `PTE_Addr = 50000`)，然后**访问物理内存地址 `50000`**，读取这个 PTE 的内容。这个 PTE 告诉 MMU，虚拟地址 `1000` 所在的虚拟页对应物理内存中的哪个物理页帧（比如说，物理页帧号是 `PFN=7`）。
- MMU 使用从 PTE 读到的信息（物理页帧号 `PFN=7`）和虚拟地址 `V_Addr` 中的页内偏移量，计算出最终的**物理地址** `P_Addr` (比如 `P_Addr = 7 * PageSize + Offset = 28672`)。
- **第二次内存访问（实际数据的访问）：** MMU 把计算出的物理地址 `P_Addr = 28672` 发送到内存总线上。
- 内存控制器从**物理内存的 `28672` 位置**取出程序真正想要的数据给 CPU。
- **总共需要两次内存访问**：一次是为了读取页表项（查找映射关系），一次是为了读取真正的数据。

**总结与例子：**

这句话的意思是，CPU 每需要访问一次虚拟内存，MMU 为了完成地址翻译，**逻辑上**需要：
1. **访问物理内存，获取页表项 (PTE)** —— 这就是所谓的“额外内存访问”。
2. **访问物理内存，获取实际数据** —— 这是程序原本就需要的访问。

**例子：**
假设：
- 页大小为 4KB ($4096$ 字节)。
- 程序想读取虚拟地址 `0x10A0` (十进制 `4256`) 的数据。
- MMU 确定这个虚拟地址属于**虚拟页 1** (`4256 / 4096 = 1`)，页内偏移是 `0x0A0` (`4256 % 4096 = 160`)。
- 操作系统已经把该进程的页表加载到了物理内存中，并且告诉了 MMU 页表的基地址。
- MMU 计算出**虚拟页 1** 对应的**页表项 (PTE)** 存储在**物理地址 `0x8004`** 处 (这个地址是 MMU 根据页表基地址和虚拟页号算出来的)。
- **额外内存访问：** MMU 向内存系统发出请求，读取**物理地址 `0x8004`** 处的数据。假设读回来的 PTE 内容表示，虚拟页 1 映射到**物理页帧 5**，并且该页有效。
- MMU 计算最终物理地址：物理页帧号 `5` * 页大小 `4096` + 页内偏移 `160` = `20480 + 160 = 20640` (即物理地址 `0x50A0`)。
- **实际数据访问：** MMU 向内存系统发出请求，读取**物理地址 `0x50A0`** 处的数据。这个数据才是程序最初想要读取的。

因此，原本程序看似一次的内存读取操作 (`read from virtual 0x10A0`)，在分页机制下，逻辑上变成了两次物理内存访问：一次访问 `0x8004` (读 PTE)，一次访问 `0x50A0` (读数据)。这就是性能开销的来源。

**重要提示：** 现代 CPU 使用一种称为 TLB (Translation Lookaside Buffer) 的高速缓存来存储最近使用过的虚拟地址到物理地址的映射。如果 MMU 在 TLB 中找到了映射关系（TLB命中），就可以跳过访问内存中的页表这一步（避免了“额外的内存访问”），从而大大降低了性能开销。只有当 TLB 未命中时，才需要真正去访问内存中的页表。所以说“逻辑上”需要额外访问，是因为这是分页机制的基础，而 TLB 是其上的优化。

### 解决思路

使用TLB
![image.png](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/20250503154815268.png)
地址转换旁路缓冲存储器（translation-lookaside buffer）

![image.png](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/20250503153807272.png)

**基本算法**
![image.png](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/20250503155322866.png)

```c++
1 VPN = (VirtualAddress & VPN_MASK) >> SHIFT 
2 (Success, TlbEntry) = TLB_Lookup(VPN) 
3 if (Success == True) // TLB Hit 
4 if (CanAccess(TlbEntry.ProtectBits) == True) 
5 Offset = VirtualAddress & OFFSET_MASK 
6 PhysAddr = (TlbEntry.PFN << SHIFT) | Offset 
7 AccessMemory(PhysAddr) 
8 else 
9 RaiseException(PROTECTION_FAULT) 
10 else // TLB Miss 
11 PTEAddr = PTBR + (VPN * sizeof(PTE)) 
12 PTE = AccessMemory(PTEAddr) 
13 if (PTE.Valid == False) 
14 RaiseException(SEGMENTATION_FAULT) 
15 else if (CanAccess(PTE.ProtectBits) == False) 
16 RaiseException(PROTECTION_FAULT) 
17 else 
18 TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits) 
19 RetryInstruction()

```
- **开始地址转换**：计算虚拟页号(VPN)
- **TLB查找**：检查TLB中是否有对应的映射
- **TLB命中分支**：
    - 检查访问权限
    - 如有权限，计算偏移、生成物理地址并访问内存
    - 如无权限，抛出保护错误(PROTECTION_FAULT)
- **TLB未命中分支**：
    - 访问页表，获取页表项(PTE)
    - 检查页表项是否有效
    - 如无效，抛出段错误(SEGMENTATION_FAULT)
    - 如有效，检查访问权限
    - 如无权限，抛出保护错误(PROTECTION_FAULT)
    - 如有权限，将映射更新到TLB并重试指令

上述系列操作开销较大，主要是因为访问页表需要额外的内存引用（第12行）​。

```c++
#include <iostream>
#include <vector> // 用于模拟 TLB 或内存 (仅为示例)
#include <optional> // 用于可能失败的查找

// --- 假设的常量和定义 (这些值依赖于具体架构) ---
// 假设 32 位地址空间, 4KB 页大小 (12 位偏移)
const uintptr_t VPN_MASK = 0xFFFFF000; // 用于提取 VPN (高 20 位)
const int SHIFT = 12;                 // 用于右移得到 VPN 或左移 PFN
const uintptr_t OFFSET_MASK = 0x00000FFF; // 用于提取页内偏移 (低 12 位)
const size_t PAGE_SIZE = 4096;
const size_t PTE_SIZE = sizeof(uintptr_t); // 简化假设：PTE 大小为一个指针宽度

// 访问类型 (用于权限检查)
enum class AccessType {
    READ,
    WRITE,
    EXECUTE
};

// 保护位 (简化示例)
// 可以用位域或者更复杂的结构
const int READ_PERMISSION = 1 << 0;
const int WRITE_PERMISSION = 1 << 1;
const int EXECUTE_PERMISSION = 1 << 2;
const int VALID_BIT = 1 << 3; // PTE 中的有效位

// 页表项 (PTE) 结构
// 实际 PTE 结构复杂得多，这里极度简化
struct PageTableEntry {
    uintptr_t data; // 简化表示，实际包含 PFN, 保护位, 有效位, 脏位, 访问位等

    bool IsValid() const {
        return (data & VALID_BIT) != 0;
    }

    int GetProtectBits() const {
        // 假设保护位存储在低位 (不含 Valid 位)
        return data & (READ_PERMISSION | WRITE_PERMISSION | EXECUTE_PERMISSION);
    }

    uintptr_t GetPFN() const {
        // 假设 PFN 存储在高位，需要屏蔽掉标志位并进行可能的对齐
        // 这是一个非常简化的例子
        return (data & VPN_MASK) >> SHIFT; // 注意：这里的 MASK/SHIFT 用于地址到 VPN/Offset，
                                           // PTE 内 PFN 的提取方式依赖于具体格式
                                           // 更准确地，PFN 应是物理地址的高位部分
                                           // uintptr_t physicalFrameAddr = data & (~OFFSET_MASK) & (~FLAGS_MASK);
                                           // return physicalFrameAddr >> SHIFT;
                                           // 为简化，我们假设 PTE 直接存 PFN << SHIFT | flags
        return (data & 0xFFFFF000) >> SHIFT; // 假设 PFN 占高 20 位
    }
};

// TLB 条目结构 (简化)
struct TlbEntry {
    uintptr_t vpn; // 虚拟页号 (用于匹配)
    uintptr_t pfn; // 物理页帧号
    int protectBits; // 保护位
    bool valid; // 条目是否有效
};

// --- 模拟的硬件/OS 功能 (桩函数) ---

// 全局变量模拟页表基址寄存器 (每个进程不同)
uintptr_t PTBR = 0x10000; // 假设页表在物理地址 0x10000 开始

// 模拟 TLB (简单数组或更复杂的数据结构)
std::vector<TlbEntry> TLB_CACHE(16); // 非常小的 TLB 示例
int tlb_replace_index = 0;

// 模拟物理内存 (非常简化)
std::vector<char> PHYSICAL_MEMORY(1024 * 1024); // 1MB 物理内存示例

struct TlbLookupResult {
    bool success;
    TlbEntry entry;
};

TlbLookupResult TLB_Lookup(uintptr_t vpn) {
    // 在 TLB_CACHE 中查找 vpn
    for (const auto& entry : TLB_CACHE) {
        if (entry.valid && entry.vpn == vpn) {
            std::cout << "[TLB] Hit for VPN: 0x" << std::hex << vpn << std::endl;
            return {true, entry};
        }
    }
    std::cout << "[TLB] Miss for VPN: 0x" << std::hex << vpn << std::endl;
    return {false, {}};
}

bool CanAccess(int protectBits, AccessType accessType) {
    std::cout << "[Check] Checking permissions..." << std::endl;
    switch (accessType) {
        case AccessType::READ:
            return (protectBits & READ_PERMISSION) != 0;
        case AccessType::WRITE:
            return (protectBits & WRITE_PERMISSION) != 0;
        case AccessType::EXECUTE:
             // 通常 Execute 隐含 Read
            return (protectBits & EXECUTE_PERMISSION) != 0;
    }
    return false;
}

// 模拟从物理内存读取一个 PTE
// 错误处理被简化
PageTableEntry ReadMemoryForPTE(uintptr_t ptePhysAddr) {
    std::cout << "[Mem] Accessing PTE at physical address: 0x" << std::hex << ptePhysAddr << std::endl;
    if (ptePhysAddr + sizeof(PageTableEntry) > PHYSICAL_MEMORY.size()) {
         std::cerr << "Error: PTE physical address out of bounds!" << std::endl;
         // 在真实系统中会触发错误
         return {0}; // 返回无效 PTE
    }
    // 简化：直接从模拟内存读取数据，假设 PTE 存储正确
    // 实际需要字节序转换等
    PageTableEntry pte;
    // 假设 PTE 的内容直接存储为 uintptr_t
    pte.data = *reinterpret_cast<uintptr_t*>(&PHYSICAL_MEMORY[ptePhysAddr]);
    return pte;
}

// 模拟最终访问数据内存 (读或写)
void AccessDataMemory(uintptr_t physAddr, AccessType accessType, uint64_t& data) {
     std::cout << "[Mem] Accessing data at final physical address: 0x" << std::hex << physAddr
               << " for " << (accessType == AccessType::READ ? "READ" : "WRITE") << std::endl;
    if (physAddr >= PHYSICAL_MEMORY.size()) {
        std::cerr << "Error: Data physical address out of bounds!" << std::endl;
        // 触发错误
        return;
    }
    // 模拟读写
    if (accessType == AccessType::READ) {
        // data = ... read from PHYSICAL_MEMORY[physAddr] ...;
        std::cout << "      (Simulated read)" << std::endl;
    } else {
        // ... write data to PHYSICAL_MEMORY[physAddr] ...;
        std::cout << "      (Simulated write)" << std::endl;
    }
}


void RaiseException(const std::string& faultType) {
    std::cerr << "[CPU] !!! Exception raised: " << faultType << " !!!" << std::endl;
    // 这里会跳转到操作系统的异常处理程序
    // 在这个模拟中，我们可能直接退出或返回错误码
    throw std::runtime_error(faultType); // 用 C++ 异常模拟
}

void TLB_Insert(uintptr_t vpn, uintptr_t pfn, int protectBits) {
    std::cout << "[TLB] Inserting/Updating entry for VPN: 0x" << std::hex << vpn
              << " -> PFN: 0x" << pfn << std::endl;
    // 使用简单的替换策略 (循环替换)
    TLB_CACHE[tlb_replace_index].vpn = vpn;
    TLB_CACHE[tlb_replace_index].pfn = pfn;
    TLB_CACHE[tlb_replace_index].protectBits = protectBits;
    TLB_CACHE[tlb_replace_index].valid = true;
    tlb_replace_index = (tlb_replace_index + 1) % TLB_CACHE.size();
}

// --- 核心翻译逻辑 ---
// 返回物理地址，如果失败则触发异常
uintptr_t TranslateVirtualToPhysical(uintptr_t virtualAddress, AccessType accessType) {
    std::cout << "\n--- Translating Virtual Address: 0x" << std::hex << virtualAddress << " ---" << std::endl;

    // 使用 goto 来模拟 RetryInstruction 的效果，虽然不推荐在常规编程中使用 goto，
    // 但它能直接反映硬件重试的流程。更好的方式是使用循环。
// retry_instruction: // 标签用于模拟重试

    // 1. 提取 VPN
    uintptr_t vpn = (virtualAddress & VPN_MASK) >> SHIFT;
    std::cout << "[CPU] Extracted VPN: 0x" << std::hex << vpn << std::endl;

    // 2. TLB 查找
    TlbLookupResult tlbResult = TLB_Lookup(vpn);

    // 3. 处理 TLB 命中
    if (tlbResult.success) {
        // 4. 检查 TLB 中的保护位
        if (CanAccess(tlbResult.entry.protectBits, accessType)) {
            // 5. 提取偏移
            uintptr_t offset = virtualAddress & OFFSET_MASK;
            // 6. 计算物理地址
            uintptr_t physAddr = (tlbResult.entry.pfn << SHIFT) | offset;
            std::cout << "[CPU] TLB Hit. Calculated PhysAddr: 0x" << std::hex << physAddr << std::endl;
            // 7. 访问内存 (模拟)
            // AccessDataMemory(physAddr, accessType, ...); // 实际访问数据的调用
            return physAddr; // 翻译成功，返回物理地址
        } else {
            // 9. TLB 命中但权限不足
             std::cout << "[CPU] TLB Hit but Protection Fault." << std::endl;
            RaiseException("PROTECTION_FAULT (from TLB)");
        }
    }
    // 10. 处理 TLB 未命中
    else {
        std::cout << "[CPU] TLB Miss. Accessing Page Table." << std::endl;
        // 11. 计算 PTE 在物理内存中的地址
        //    注意：这假设是单级页表。多级页表会更复杂，需要多次内存访问。
        uintptr_t pteAddr = PTBR + (vpn * sizeof(PageTableEntry)); // 简化假设：PTE 大小为结构大小
                                                                  // 更实际的是用 PTE_SIZE 宏

        // 12. 访问物理内存以获取 PTE (!!! 额外的内存访问 !!!)
        PageTableEntry pte = ReadMemoryForPTE(pteAddr);
         std::cout << "[CPU] Fetched PTE data: 0x" << std::hex << pte.data << std::endl;


        // 13. 检查 PTE 是否有效
        if (!pte.IsValid()) {
            std::cout << "[CPU] PTE invalid." << std::endl;
            RaiseException("SEGMENTATION_FAULT (Invalid PTE)");
        }
        // 15. 检查 PTE 中的保护位
        else if (!CanAccess(pte.GetProtectBits(), accessType)) {
             std::cout << "[CPU] PTE valid but Protection Fault." << std::endl;
            RaiseException("PROTECTION_FAULT (from PTE)");
        }
        // 17. PTE 有效且权限允许
        else {
             std::cout << "[CPU] PTE valid and access permitted." << std::endl;
            // 18. 将映射关系插入 TLB
            TLB_Insert(vpn, pte.GetPFN(), pte.GetProtectBits());

            // 19. 重试指令 (模拟)
            // 在真实硬件中，CPU 会重新执行导致此过程的指令。
            // 此时因为 TLB 已经被填充，下一次尝试应该会 TLB 命中。
            // 这里我们通过再次计算物理地址并返回来模拟成功路径。
             std::cout << "[CPU] Retrying instruction (simulated by recalculating PhysAddr after TLB insert)." << std::endl;
            // goto retry_instruction; // 可以用 goto 跳转回开头重新查找 TLB

            // 或者直接计算并返回，因为我们知道下次会命中
            uintptr_t offset = virtualAddress & OFFSET_MASK;
            uintptr_t physAddr = (pte.GetPFN() << SHIFT) | offset;
             std::cout << "[CPU] Calculated PhysAddr after TLB miss handling: 0x" << std::hex << physAddr << std::endl;
             // AccessDataMemory(physAddr, accessType, ...); // 实际访问数据的调用
             return physAddr;
        }
    }
    // 通常不会执行到这里，因为所有路径要么返回地址，要么抛出异常
    return 0; // 表示错误或未处理的情况
}

// --- 主函数示例 ---
int main() {
    // 示例：设置一个有效的 PTE (手动放入模拟内存)
    uintptr_t exampleVPN = 0x1; // 访问虚拟页 1
    uintptr_t targetPTEAddr = PTBR + (exampleVPN * sizeof(PageTableEntry));
    uintptr_t targetPFN = 0x5; // 假设映射到物理页帧 5
    int permissions = READ_PERMISSION | WRITE_PERMISSION | VALID_BIT;
    PageTableEntry examplePTE;
    examplePTE.data = (targetPFN << SHIFT) | permissions; // 构造 PTE 内容

    if (targetPTEAddr + sizeof(PageTableEntry) <= PHYSICAL_MEMORY.size()) {
         std::cout << "Setting up PTE for VPN 0x1 at physical address 0x" << std::hex << targetPTEAddr << std::endl;
        *reinterpret_cast<uintptr_t*>(&PHYSICAL_MEMORY[targetPTEAddr]) = examplePTE.data;
    } else {
        std::cerr << "Cannot setup example PTE, address out of bounds." << std::endl;
        return 1;
    }


    try {
        // 第一次访问，应该 TLB Miss，然后从内存加载 PTE，填充 TLB，成功返回
        uintptr_t virtualAddr1 = (exampleVPN << SHIFT) | 0x0A0; // 虚拟地址 0x10A0
        std::cout << "Attempting first access (expect TLB Miss then Success)..." << std::endl;
        uintptr_t physAddr1 = TranslateVirtualToPhysical(virtualAddr1, AccessType::READ);
        std::cout << "==> Success! Virtual 0x" << std::hex << virtualAddr1 << " -> Physical 0x" << physAddr1 << std::endl;
        // 模拟实际内存操作
        uint64_t readData; // 假设读取 64 位数据
        AccessDataMemory(physAddr1, AccessType::READ, readData);


        // 第二次访问同一页，应该 TLB Hit
        uintptr_t virtualAddr2 = (exampleVPN << SHIFT) | 0x0B0; // 虚拟地址 0x10B0
        std::cout << "\nAttempting second access (expect TLB Hit)..." << std::endl;
        uintptr_t physAddr2 = TranslateVirtualToPhysical(virtualAddr2, AccessType::WRITE);
         std::cout << "==> Success! Virtual 0x" << std::hex << virtualAddr2 << " -> Physical 0x" << physAddr2 << std::endl;
         uint64_t writeData = 0; // 假设写入 64 位数据
        AccessDataMemory(physAddr2, AccessType::WRITE, writeData);

        // 尝试访问一个无效的页 (假设 VPN=0x2 的 PTE 不存在或无效)
         uintptr_t virtualAddr3 = (0x2 << SHIFT) | 0x0C0; // 虚拟地址 0x20C0
         std::cout << "\nAttempting access to unmapped page (expect SegFault)..." << std::endl;
         TranslateVirtualToPhysical(virtualAddr3, AccessType::READ); // 应该抛出异常


    } catch (const std::runtime_error& e) {
        std::cerr << "\nCaught expected exception: " << e.what() << std::endl;
    } catch (...) {
         std::cerr << "\nCaught unexpected exception." << std::endl;
    }

    return 0;
}
```

**代码解释和要点：**

1. **结构体定义:** 定义了 `PageTableEntry` 和 `TlbEntry` 来模拟页表项和 TLB 条目，包含必要的信息（PFN、保护位、有效位等）。这些是高度简化的。
2. **常量:** 定义了 `VPN_MASK`, `SHIFT`, `OFFSET_MASK` 等常量，它们的值依赖于假定的内存架构（例如 32 位地址、4KB 页）。
3. **桩函数:** `TLB_Lookup`, `CanAccess`, `ReadMemoryForPTE`, `AccessDataMemory`, `RaiseException`, `TLB_Insert` 模拟了硬件或 OS 底层的功能。它们只打印信息或进行非常简单的模拟操作。
    - `ReadMemoryForPTE` 特别重要，它模拟了 **读取物理内存以获取 PTE** 的过程，这就是导致额外开销的地方。
    - `AccessDataMemory` 模拟最终对计算出的物理地址进行数据读写。
    - `RaiseException` 使用 C++ 异常来模拟硬件/OS 异常。
    - `TLB_Lookup` 和 `TLB_Insert` 使用一个简单的 `std::vector` 来模拟 TLB 缓存。

4. **`TranslateVirtualToPhysical` 函数:** 这是核心逻辑，严格按照伪代码的步骤实现：
    - 提取 VPN。
    - 查找 TLB。
    - **TLB 命中:** 检查权限，计算物理地址，返回。如果权限错误，抛出异常。
    - **TLB 未命中:** 计算 PTE 地址，**访问内存获取 PTE**，检查 PTE 有效性，检查权限。如果无效或权限错误，抛出异常。
    - **成功获取 PTE:** 将信息插入 TLB，计算物理地址，返回（模拟重试后的成功路径）。
5. **重试逻辑 (`RetryInstruction`)**: 在这个 C++ 代码中，通过在 TLB 插入后直接计算并返回物理地址来模拟重试成功的效果。也可以使用 `goto` 跳转回函数开头（如注释所示）来更直接地模拟重试查找 TLB 的过程，但这通常不被认为是良好的 C++ 风格。

### 测试

```c
// File: tlb.c
#include <stdio.h>      // 标准输入输出库
#include <stdlib.h>     // 标准库 (包含 atol, calloc, free, exit 等)
#include <time.h>       // 时间相关函数 (clock_gettime, struct timespec)
#include <unistd.h>     // POSIX 操作系统 API (包含 sysconf)
#include <stdint.h>     // 定义精确宽度的整数类型 (如 uint64_t)

// 定义每秒包含的纳秒数 (用于时间转换)
#define NSECS_PER_SEC 1000000000ULL // ULL 表示无符号长长整型常量

// 辅助函数：获取当前时间的纳秒表示 (使用单调时钟)
// CLOCK_MONOTONIC 提供了一个从某个固定点开始单调递增的时间，不受系统时钟调整的影响
static inline uint64_t get_time_ns() {
    struct timespec ts; // 定义 timespec 结构体变量，用于存储秒和纳秒
    // 调用 clock_gettime 获取 CLOCK_MONOTONIC 时间，并存入 ts
    if (clock_gettime(CLOCK_MONOTONIC, &ts) == 0) { // 检查调用是否成功 (返回 0 表示成功)
        // 成功，则计算总纳秒数 = 秒数 * 每秒纳秒数 + 纳秒数
        return (uint64_t)ts.tv_sec * NSECS_PER_SEC + (uint64_t)ts.tv_nsec;
    } else {
        // 调用失败，打印错误信息 (perror 会根据 errno 输出具体的错误原因)
        perror("clock_gettime");
        // 退出程序，并返回失败状态码
        exit(EXIT_FAILURE);
    }
}

int main(int argc, char *argv[]) {
    // 检查命令行参数数量是否正确 (程序名 + 2个参数 = 3)
    if (argc != 3) {
        // 如果参数数量不对，向标准错误输出用法提示
        fprintf(stderr, "Usage: %s <num_pages> <num_trials>\n", argv[0]);
        // 返回失败状态码
        return EXIT_FAILURE;
    }

    // 解析命令行参数
    // atol 函数将字符串转换为 long 类型整数
    long num_pages = atol(argv[1]);  // 获取用户输入的页数
    long num_trials = atol(argv[2]); // 获取用户输入的试验次数

    // 检查输入的页数和试验次数是否为正数
    if (num_pages <= 0 || num_trials <= 0) {
        fprintf(stderr, "Error: Number of pages and trials must be positive.\n");
        return EXIT_FAILURE;
    }

    // 获取系统物理内存页的大小 (以字节为单位)
    // sysconf(_SC_PAGESIZE) 是一个 POSIX 标准调用，用于查询系统配置信息
    long page_size = sysconf(_SC_PAGESIZE);
    if (page_size < 0) { // 如果 sysconf 调用失败，会返回 -1
        perror("sysconf(_SC_PAGESIZE)");
        return EXIT_FAILURE;
    }
    // 打印系统页大小、用户指定的页数和试验次数
    printf("System Page Size: %ld bytes\n", page_size);
    printf("Number of Pages to Access: %ld\n", num_pages);
    printf("Number of Trials: %ld\n", num_trials);

    // 计算需要分配的总内存大小
    // 使用 size_t 类型进行计算，以避免在处理大内存时发生整数溢出
    size_t total_size = (size_t)num_pages * (size_t)page_size;
    // 打印将要分配的总内存大小（字节和兆字节 MB）
    printf("Total memory to allocate: %zu bytes (%.2f MB)\n",
           total_size, (double)total_size / (1024.0 * 1024.0)); // %zu 是 size_t 的格式说明符

    // 分配内存
    // 使用 calloc 分配内存，它会将分配的内存初始化为零。
    // 这有助于确保在访问前，操作系统已经为这些虚拟地址分配了物理页框并建立了映射（或至少准备好了按需映射）。
    // 使用 char* 类型指针，这样可以通过 buffer[i] 方便地访问任意字节。
    char *buffer = (char *)calloc(total_size, 1);
    if (buffer == NULL) { // 检查内存分配是否成功
        perror("Failed to allocate memory");
        return EXIT_FAILURE;
    }

    // --- 在正式计时循环开始前，先"预热"，访问每个页的第一个字节 ---
    // 目的是尽量触发所有必要的缺页中断（Page Fault），让操作系统完成虚拟页到物理页的初始映射。
    // 这样可以减少第一次计时试验中包含大量初始映射开销，使得后续计时更稳定地反映 TLB 和缓存行为。
    printf("Pre-touching pages...\n");
    for (long i = 0; i < num_pages; ++i) {
         // 计算第 i 个页的起始地址索引
         size_t index = (size_t)i * (size_t)page_size;
         // 进行边界检查，确保索引不会越界 (虽然理论上按计算逻辑不会，但这是好习惯)
         if (index < total_size) {
             // 对每个页的第一个字节执行一个简单的写操作 (自增)
             // 这个写操作会确保该地址被访问，从而触发缺页处理（如果需要）
             buffer[index]++;
         }
    }
    printf("Pre-touching complete.\n");


    uint64_t total_elapsed_ns = 0; // 用于累加所有试验的总耗时（纳秒）
    // 定义一个 volatile 变量。volatile 告诉编译器不要对这个变量的读写进行优化（例如，不要缓存到寄存器中或删除看似无用的读写）。
    // 这里用它来接收内存读取的值（虽然在本例中没实际使用读取的值），可以帮助确保内存访问确实发生，不被编译器优化掉。
    volatile char touch;

    printf("Starting timing trials...\n"); // 提示计时开始

    // 主要的计时循环，共执行 num_trials 次试验
    for (long trial = 0; trial < num_trials; ++trial) {
        uint64_t start_ns = get_time_ns(); // 记录本次试验循环的开始时间

        // 内部循环：按顺序访问每个页
        for (long i = 0; i < num_pages; ++i) {
            // 计算第 i 个页的起始字节的索引
            size_t index = (size_t)i * (size_t)page_size;

            // 在每个页内执行一个简单的内存访问操作
            // 使用自增操作 `++` 给内存访问增加“副作用”，进一步防止编译器认为访问无用而将其优化掉。
             if (index < total_size) { // 边界检查
                 buffer[index]++; // 访问内存（执行写操作）
                 // 可选：可以加上一步读取操作，赋值给 volatile 变量，进一步确保访问发生
                 // touch = buffer[index];
             }
        }

        uint64_t end_ns = get_time_ns(); // 记录本次试验循环的结束时间
        total_elapsed_ns += (end_ns - start_ns); // 将本次试验耗时累加到总耗时中

        // 可选：可以取消注释下面的代码，以打印每次试验或每隔几次试验的进度
        // if ((trial + 1) % 10 == 0 || trial == num_trials - 1) {
        //     printf("Trial %ld completed.\n", trial + 1);
        // }
    }

    printf("Timing trials complete.\n"); // 提示所有计时试验完成

    // 释放之前通过 calloc 分配的内存
    free(buffer);

    // 计算并打印最终的测量结果
    if (num_trials > 0 && num_pages > 0) { // 进行有效性检查，防止除以零
        // 计算平均每次试验的总耗时（纳秒）
        double avg_trial_time_ns = (double)total_elapsed_ns / num_trials;
        // 计算平均每次访问一个页所花费的时间（纳秒）
        double avg_page_access_time_ns = avg_trial_time_ns / num_pages;

        printf("\n--- Results ---\n"); // 打印结果分隔符
        // 打印所有试验的总耗时（转换为毫秒，更直观）
        printf("Total elapsed time for %ld trials: %.3f ms\n",
               num_trials, (double)total_elapsed_ns / 1000000.0); // 1毫秒 = 1,000,000纳秒
        // 打印平均每次试验的耗时（纳秒）
        printf("Average time per trial (accessing %ld pages): %.3f ns\n",
               num_pages, avg_trial_time_ns);
        // 打印最终的核心结果：平均每次访问一个页的耗时（纳秒）
        printf("Average time per page access: %.3f ns\n", avg_page_access_time_ns);
    } else {
        // 如果试验次数或页数为 0，则不进行计算和打印结果
        printf("\nNo trials run or zero pages specified.\n");
    }


    return EXIT_SUCCESS; // 程序正常结束，返回成功状态码
}


```

![image.png](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/20250503155758404.png)
**运行 1:**
- `./tlb 1024 100`
- 访问页数：1024
- 试验次数：100
- 总内存：4 MB
- **平均每次访问页的耗时：4.662 纳秒 (ns)**
**运行 2:**
- `./tlb 131072 10`
- 访问页数：131072
- 试验次数：10
- 总内存：512 MB
- **平均每次访问页的耗时：11.620 纳秒 (ns)**
**结果分析:**
最关键的发现是，当访问的页数从 1024 (4MB 内存) 大幅增加到 131072 (512MB 内存) 时，**平均每次访问一个页所需的时间显著增加了** (从 4.662 ns 增加到 11.620 ns)。

**原因解释:**
1. **小工作集 (1024 页):**
    - **TLB 命中率高:** 1024 个页所需的地址翻译条目数量相对较少。现代 CPU 的 TLB 通常能缓存几百到几千个页的映射。因此，在访问这 1024 个页时，地址翻译很可能大部分时间都能在 TLB 中找到（TLB Hit），避免了访问内存中的页表（Page Walk）这一较慢的操作。
    - **缓存命中率高:** 4MB 的数据量也可能很好地适应 CPU 的 L2 或 L3 缓存。这意味着访问的数据本身也经常能从快速缓存中获取，而不是慢速的主内存。
    - **结果:** 高 TLB 命中率和高缓存命中率共同作用，使得平均访问时间非常短（约 4.66 ns）。
        
2. **大工作集 (131072 页):**
    - **TLB 命中率降低:** 131072 个页远远超过了典型 TLB 的容量。当程序依次访问这些页时，TLB 需要不断地替换旧的条目，加载新的条目。这导致了大量的 TLB Miss。每次 TLB Miss 都需要硬件去访问内存中的页表（Page Table）来查找物理地址，这个过程比 TLB Hit 慢得多。
    - **缓存命中率降低:** 512MB 的数据量很可能超出了 CPU 的所有层级缓存（L1, L2, L3）。因此，不仅地址翻译变慢（TLB Miss 增多），数据本身的访问也更频繁地需要从主内存读取，而不是从快速缓存读取。
    - **结果:** TLB Miss 导致的 Page Walk 开销和缓存 Miss 导致的内存访问开销叠加，使得平均访问时间显著增加（约 11.62 ns）。

**结论:**
- 访问内存的性能**不是恒定的**。
- 当程序访问的内存范围（工作集大小）超出了 TLB 和 CPU 缓存的容量时，由于 TLB Miss 和 Cache Miss 的增加，平均内存访问时间会**显著变慢**。
- TLB 和缓存是提高虚拟内存性能的关键机制，保持程序的工作集尽可能小以适应这些硬件缓存，对于性能优化至关重要。


还有一个需要注意的地方，今天的计算机系统大多有多个CPU，每个CPU当然有自己的TLB结构。为了得到准确的测量数据，我们需要只在一个CPU上运行程序，避免调度器把进程从一个CPU调度到另一个去运行。如何做到？​（提示：在Google上搜索“pinning a thread”相关的信息）如果没有这样做，代码从一个CPU移到了另一个，会发生什么情况？