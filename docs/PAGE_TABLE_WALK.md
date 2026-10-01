# 单级页表地址转换

本练习位于 `exercises/06_page_table/02_page_table_walk/src/lib.rs`，用一个
简化的单级页表模拟虚拟地址到物理地址的转换过程。

页表保存虚拟页号（VPN）到物理页号（PPN）的映射。转换时，硬件或模拟器先
查找虚拟页对应的页表项，再将物理页号与原虚拟地址中的页内偏移组合成物理
地址。

## 1. 地址格式

本练习使用 32 位虚拟地址和 4 KB 页大小：

```text
31                         12 11                          0
+----------------------------+-----------------------------+
|          VPN (20 位)        |       页内偏移 (12 位)       |
+----------------------------+-----------------------------+
```

```rust
pub const PAGE_SIZE: usize = 4096;
pub const PAGE_OFFSET_BITS: u32 = 12;
```

由于：

```text
4 KB = 4096 bytes = 2^12 bytes
```

每个虚拟地址的低 12 位表示页内偏移，高 20 位表示虚拟页号。

## 2. 拆分虚拟地址

### 提取 VPN

```rust
pub fn va_to_vpn(va: u32) -> usize {
    (va >> PAGE_OFFSET_BITS) as usize
}
```

右移 12 位去掉页内偏移，留下高位 VPN。

### 提取页内偏移

```rust
pub fn va_to_offset(va: u32) -> u32 {
    va & ((1 << PAGE_OFFSET_BITS) - 1)
}
```

掩码的低 12 位为 1，其余位为 0，因此按位与后只保留页内偏移。

例如，虚拟地址 `0x12345678`：

```text
VPN    = 0x12345678 >> 12 = 0x12345
offset = 0x12345678 & 0xFFF = 0x678
```

## 3. 页表项

```rust
pub struct PageTableEntry {
    pub ppn: u32,
    pub flags: u8,
}
```

- `ppn`：虚拟页映射到的物理页号。
- `flags`：页表项状态和访问权限。

练习定义了三个标志：

```rust
pub const PTE_VALID: u8 = 1 << 0;
pub const PTE_READ: u8 = 1 << 1;
pub const PTE_WRITE: u8 = 1 << 2;
```

分别表示页表项有效、可读和可写。当前翻译接口只接受 `is_write` 参数，
因此只检查有效位和写权限；虽然定义了 `PTE_READ`，本练习没有单独实现读
权限请求检查。

## 4. 单级页表结构

```rust
pub struct SingleLevelPageTable {
    entries: Vec<Option<PageTableEntry>>,
}
```

`entries` 的索引就是 VPN：

```text
entries[vpn] = Some(PTE)  表示该虚拟页有映射
entries[vpn] = None       表示该虚拟页未映射
```

创建页表时指定可支持的虚拟页数量：

```rust
let mut page_table = SingleLevelPageTable::new(1024);
```

这表示页表可以存放索引 `0..1024` 的 VPN 映射。

## 5. 建立、查询和移除映射

### `map`

```rust
page_table.map(vpn, ppn, flags);
```

`map` 在 `entries[vpn]` 中保存新的 `PageTableEntry`。如果该 VPN 已有映射，
新页表项会替换旧页表项。

当前实现通过索引直接访问 `entries[vpn]`。调用者需要保证 `vpn` 在页表
范围内，否则会触发越界 panic。

### `lookup`

```rust
pub fn lookup(&self, vpn: usize) -> Option<&PageTableEntry> {
    self.entries.get(vpn)?.as_ref()
}
```

`get(vpn)` 会检查索引是否在范围内：

- VPN 越界时返回 `None`。
- VPN 在范围内但没有映射时返回 `None`。
- 找到映射时返回 `Some(&PageTableEntry)`。

因此，查询未映射或越界 VPN 都不会导致 panic。

### `unmap`

```rust
page_table.unmap(vpn);
```

它将 `entries[vpn]` 设为 `None`，表示移除该映射。当前实现同样要求 `vpn`
在页表范围内。

## 6. 地址翻译流程

```rust
pub fn translate(&self, va: u32, is_write: bool) -> TranslateResult
```

转换过程按以下顺序执行：

```text
虚拟地址 va
    |
    +-- 提取 VPN 和 offset
    |
    +-- 用 VPN 查询页表
    |       未映射或越界 -> PageFault
    |
    +-- 检查 PTE_VALID
    |       未设置 -> PageFault
    |
    +-- 若请求写访问，检查 PTE_WRITE
    |       未设置 -> PermissionDenied
    |
    +-- 组合物理地址
            返回 Ok(pa)
```

对应的实现逻辑：

```rust
let vpn = va_to_vpn(va);
let offset = va_to_offset(va);

let Some(entry) = self.lookup(vpn) else {
    return TranslateResult::PageFault;
};

if entry.flags & PTE_VALID == 0 {
    return TranslateResult::PageFault;
}

if is_write && entry.flags & PTE_WRITE == 0 {
    return TranslateResult::PermissionDenied;
}

TranslateResult::Ok(make_pa(entry.ppn, offset))
```

先判断映射是否存在和有效，再检查访问权限。这样无效页返回缺页结果，
只有有效映射才会进入权限检查。

## 7. 翻译结果

```rust
pub enum TranslateResult {
    Ok(u32),
    PageFault,
    PermissionDenied,
}
```

- `Ok(pa)`：翻译成功，包含物理地址。
- `PageFault`：VPN 未映射、VPN 超出页表范围，或 PTE 的有效位未设置。
- `PermissionDenied`：映射有效，但写访问没有写权限。

这几种结果区分了“地址没有有效映射”和“映射存在但访问不允许”。

## 8. 检查写权限

```rust
if is_write && entry.flags & PTE_WRITE == 0 {
    return TranslateResult::PermissionDenied;
}
```

只有调用者请求写入时，才需要检查 `PTE_WRITE`：

| `is_write` | PTE_WRITE | 结果 |
|---|---|---|
| `false` | 未设置 | 继续翻译 |
| `false` | 已设置 | 继续翻译 |
| `true` | 已设置 | 继续翻译 |
| `true` | 未设置 | `PermissionDenied` |

读访问不会因为页面带有额外的写权限而失败。当前练习没有读访问参数，
因此没有检查 `PTE_READ`。

## 9. 组合物理地址

```rust
pub fn make_pa(ppn: u32, offset: u32) -> u32 {
    ppn * PAGE_SIZE as u32 + offset
}
```

物理页号表示页帧编号，乘以页大小即可得到页帧起始物理地址。然后加上原
虚拟地址中的页内偏移：

```text
物理地址 = PPN × PAGE_SIZE + offset
```

示例：

```text
PPN    = 0x80
offset = 0x100

PA = 0x80 × 4096 + 0x100
   = 0x80100
```

页内偏移在映射过程中保持不变。

## 10. 完整示例

将虚拟页 `1` 映射到物理页 `0x80`，并设置有效和可读标志：

```rust
let mut page_table = SingleLevelPageTable::new(1024);
page_table.map(1, 0x80, PTE_VALID | PTE_READ);
```

翻译虚拟地址 `0x1100`：

```text
VPN    = 0x1100 >> 12 = 1
offset = 0x1100 & 0xFFF = 0x100
```

VPN `1` 映射到 PPN `0x80`，页表项有效，且本次不是写访问，因此：

```text
PA = 0x80 × 4096 + 0x100 = 0x80100
```

结果为：

```rust
TranslateResult::Ok(0x80100)
```

如果对同一页执行写访问，由于该 PTE 没有 `PTE_WRITE`，结果为：

```rust
TranslateResult::PermissionDenied
```

## 11. 与真实 SV39 的区别

本练习注释中的地址格式是 32 位简化模型，页表只有一个 `Vec`，通过 VPN
直接索引 PTE。

真实 RISC-V SV39 使用 39 位虚拟地址和三级页表：

```text
VPN[2] -> VPN[1] -> VPN[0] -> 页内偏移
```

页表遍历时需要逐级读取页表项，并区分中间级页表项和最终叶子页表项。
本练习省略了多级遍历、用户权限、执行权限、全局映射、访问位、脏位和大页
等机制，重点演示 VPN/offset 拆分、映射查找、有效性检查、写权限检查和物理
地址组合。

## 12. 数值范围注意事项

练习使用 `u32` 表示虚拟地址、PPN 和物理地址，便于演示。真实系统需要根据
物理地址宽度和架构规则处理范围。

`make_pa` 使用 `u32` 乘法和加法。如果传入过大的 PPN，结果可能超出 `u32`
可表示范围。当前测试使用的数值都在范围内；生产级实现应明确地址宽度并
处理溢出或非法 PPN。

## 13. 核心要点

1. 4 KB 页大小对应 12 位页内偏移。
2. VPN 用于查询页表，offset 在地址翻译中保持不变。
3. 页表项将 VPN 映射到 PPN，并通过标志位描述有效性和权限。
4. 未映射、越界或无效 PTE 返回 `PageFault`。
5. 写访问缺少 `PTE_WRITE` 时返回 `PermissionDenied`。
6. 物理地址通过 `PPN × PAGE_SIZE + offset` 计算。
7. 当前练习是单级教学模型，不等同于真实 SV39 多级页表遍历。
