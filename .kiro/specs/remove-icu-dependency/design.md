# Remove ICU Dependency Bugfix Design

## Overview

本修复旨在解决 Windows Server 2016 环境下因缺少 icuuc.dll 文件导致应用程序无法启动的问题。根本原因是 `url = "2.5"` crate 默认启用了 `idna` 特性，该特性通过依赖链引入了 ICU 库。修复策略是禁用 url crate 的默认特性，仅启用必要的 `serde` 特性，从而移除 ICU 依赖。

由于项目实际使用的是 `reqwest::Url`（reqwest 重导出的 url crate），而非直接使用国际化域名（IDNA）功能，因此禁用 idna 特性不会影响现有功能。

## Glossary

- **Bug_Condition (C)**: 触发 bug 的条件 - 当应用程序在 Windows Server 2016 上启动时，系统找不到 icuuc.dll 文件
- **Property (P)**: 期望的行为 - 应用程序应该在没有外部 ICU DLL 的情况下正常启动和运行
- **Preservation**: 必须保持不变的现有行为 - URL 解析功能、序列化功能、其他依赖项的正常工作
- **idna 特性**: url crate 的一个可选特性，用于支持国际化域名（Internationalized Domain Names in Applications），会引入 ICU 依赖
- **ICU (International Components for Unicode)**: Unicode 国际化组件库，在某些 Windows 环境下需要额外的 DLL 文件
- **reqwest::Url**: reqwest crate 重导出的 url::Url 类型，项目中实际使用的 URL 解析类型

## Bug Details

### Fault Condition

当应用程序在 Windows Server 2016 环境上启动时，由于 url crate 的默认配置启用了 idna 特性，导致依赖链引入了 ICU 库（idna → idna_adapter → icu_normalizer/icu_properties），而 ICU 库需要 icuuc.dll 等外部 DLL 文件，这些文件在 Windows Server 2016 环境中不存在或不兼容。

**Formal Specification:**
```
FUNCTION isBugCondition(input)
  INPUT: input of type RuntimeEnvironment
  OUTPUT: boolean
  
  RETURN input.os == "Windows Server 2016"
         AND input.hasFile("icuuc.dll") == false
         AND application.dependencies.contains("url crate with idna feature")
         AND application.startupFails == true
END FUNCTION
```

### Examples

- **示例 1**: 在 Windows Server 2016 上启动应用程序
  - 预期行为: 应用程序正常启动
  - 实际行为: 系统提示 "找不到 icuuc.dll" 并无法启动

- **示例 2**: 使用 `url = "2.5"` 默认配置
  - 预期行为: 仅引入必要的 URL 解析功能
  - 实际行为: 自动启用 idna 特性，引入完整的 ICU 依赖链

- **示例 3**: 解析标准 URL（如 "https://example.com/path"）
  - 预期行为: 正常解析 URL
  - 实际行为: 功能正常，但携带了不必要的 ICU 依赖

- **边缘情况**: 在其他 Windows 环境（Windows 10/11）上运行
  - 预期行为: 应用程序继续正常工作

## Expected Behavior

### Preservation Requirements

**Unchanged Behaviors:**
- 标准 URL 解析功能必须继续正常工作（如解析 "https://example.com/path"）
- URL 的序列化功能（serde）必须继续正常工作
- reqwest 等依赖项使用 url crate 的功能必须继续正常工作
- 在其他 Windows 环境（Windows 10、Windows 11）上的运行必须继续正常

**Scope:**
所有不涉及国际化域名（IDNA）的 URL 操作都应该完全不受此修复影响。这包括:
- 标准 ASCII 域名的 URL 解析
- URL 的序列化和反序列化
- reqwest HTTP 客户端的 URL 处理
- WebDAV 同步功能中的 URL 验证

## Hypothesized Root Cause

基于 bug 描述和依赖分析，最可能的问题是:

1. **默认特性启用**: url crate 的默认配置启用了 idna 特性
   - `url = "2.5"` 等同于 `url = { version = "2.5", default-features = true }`
   - 默认特性包含 idna，用于支持国际化域名

2. **依赖链传递**: idna 特性引入了完整的 ICU 依赖链
   - idna → idna_adapter → icu_normalizer
   - idna → idna_adapter → icu_properties
   - 这些 crate 依赖于 ICU 库的本地实现

3. **Windows Server 2016 兼容性**: ICU 库在 Windows Server 2016 上存在兼容性问题
   - 缺少必要的 DLL 文件（icuuc.dll）
   - 或者 DLL 版本不兼容

4. **功能未使用**: 项目实际上不需要 IDNA 功能
   - 代码中使用的是 `reqwest::Url::parse()`
   - 解析的都是标准 ASCII 域名，不涉及国际化域名

## Correctness Properties

Property 1: Fault Condition - Application Starts Without ICU DLL

_For any_ Windows Server 2016 环境，当应用程序启动时，修复后的应用程序 SHALL 能够正常启动和运行，而不依赖外部的 icuuc.dll 文件或其他 ICU 相关 DLL。

**Validates: Requirements 2.1, 2.2**

Property 2: Preservation - URL Parsing Functionality

_For any_ 标准 URL 输入（非国际化域名），修复后的 url crate SHALL 产生与原始配置完全相同的解析结果，保持所有现有的 URL 解析、序列化和相关功能正常工作。

**Validates: Requirements 3.1, 3.2, 3.3, 3.4**

## Fix Implementation

### Changes Required

假设我们的根本原因分析是正确的:

**File**: `src/Cargo.toml`

**Section**: `[dependencies]`

**Specific Changes**:
1. **禁用 url crate 的默认特性**: 将 `url = "2.5"` 修改为显式配置
   - 修改为: `url = { version = "2.5", default-features = false, features = ["serde"] }`
   - 这将禁用 idna 特性，移除 ICU 依赖链

2. **保留 serde 特性**: 确保 URL 序列化功能继续可用
   - serde 特性是项目所需的，用于 URL 的序列化和反序列化
   - 这个特性不依赖 ICU 库

3. **验证依赖树**: 修改后需要验证 ICU 相关依赖已被移除
   - 使用 `cargo tree` 检查依赖树
   - 确认不再包含 icu_normalizer、icu_properties 等 crate

4. **测试编译**: 确保项目能够正常编译
   - 运行 `cargo build` 验证编译成功
   - 检查是否有编译错误或警告

5. **功能验证**: 确保现有功能不受影响
   - 测试 WebDAV URL 解析功能
   - 测试 reqwest HTTP 请求功能
   - 验证 URL 序列化功能

## Testing Strategy

### Validation Approach

测试策略采用两阶段方法：首先在未修复的代码上验证 bug 的存在（在 Windows Server 2016 环境），然后验证修复后的代码能够正常工作并保持现有功能不变。

### Exploratory Fault Condition Checking

**Goal**: 在实施修复之前，在 Windows Server 2016 环境上验证 bug 的存在。确认或反驳根本原因分析。如果反驳，需要重新假设。

**Test Plan**: 在 Windows Server 2016 环境上尝试启动未修复的应用程序，观察是否出现 icuuc.dll 缺失错误。同时使用 `cargo tree` 检查依赖树，确认 ICU 依赖的存在。

**Test Cases**:
1. **Windows Server 2016 启动测试**: 在 Windows Server 2016 上启动应用程序（未修复代码将失败）
2. **依赖树检查**: 运行 `cargo tree | grep -i icu` 查看 ICU 相关依赖（未修复代码将显示 icu_normalizer 等）
3. **DLL 依赖检查**: 使用工具（如 Dependency Walker）检查编译后的可执行文件的 DLL 依赖（未修复代码将显示 icuuc.dll）
4. **其他 Windows 环境测试**: 在 Windows 10/11 上启动应用程序（可能成功，取决于 ICU DLL 的可用性）

**Expected Counterexamples**:
- 应用程序在 Windows Server 2016 上启动失败，提示找不到 icuuc.dll
- 可能的原因: url crate 默认启用 idna 特性，ICU 库不兼容，DLL 文件缺失

### Fix Checking

**Goal**: 验证对于所有触发 bug 条件的输入（Windows Server 2016 环境），修复后的应用程序能够正常启动。

**Pseudocode:**
```
FOR ALL environment WHERE isBugCondition(environment) DO
  result := startApplication_fixed(environment)
  ASSERT result.started == true
  ASSERT result.error == null
  ASSERT NOT result.requiresExternalDLL("icuuc.dll")
END FOR
```

### Preservation Checking

**Goal**: 验证对于所有不触发 bug 条件的输入（标准 URL 解析操作），修复后的代码产生与原始代码相同的结果。

**Pseudocode:**
```
FOR ALL url_input WHERE NOT isBugCondition(url_input) DO
  ASSERT parseUrl_original(url_input) = parseUrl_fixed(url_input)
END FOR
```

**Testing Approach**: 属性测试（Property-based testing）推荐用于保持性检查，因为:
- 它自动生成大量测试用例覆盖输入域
- 它能捕获手动单元测试可能遗漏的边缘情况
- 它提供强有力的保证，确保所有非 bug 输入的行为保持不变

**Test Plan**: 首先在未修复的代码上观察标准 URL 解析的行为，然后编写属性测试捕获该行为，验证修复后行为一致。

**Test Cases**:
1. **标准 URL 解析保持性**: 验证解析标准 URL（如 "https://example.com/path"）的结果保持不变
2. **URL 序列化保持性**: 验证 URL 的序列化和反序列化功能继续正常工作
3. **reqwest 集成保持性**: 验证 reqwest HTTP 客户端的 URL 处理继续正常工作
4. **WebDAV URL 验证保持性**: 验证 WebDAV 同步功能中的 URL 验证继续正常工作

### Unit Tests

- 测试标准 URL 解析功能（ASCII 域名）
- 测试 URL 序列化和反序列化
- 测试边缘情况（空 URL、特殊字符、长 URL）
- 测试 WebDAV 配置中的 URL 验证

### Property-Based Tests

- 生成随机的标准 URL 并验证解析结果一致性
- 生成随机的 URL 配置并验证序列化/反序列化的往返一致性
- 测试大量 URL 输入场景，确保修复后行为保持不变

### Integration Tests

- 在 Windows Server 2016 环境上测试完整的应用程序启动流程
- 测试 WebDAV 同步功能的完整流程（包括 URL 解析和 HTTP 请求）
- 测试在不同 Windows 环境（Windows 10、Windows 11）上的兼容性
- 验证应用程序不再依赖外部 ICU DLL 文件
