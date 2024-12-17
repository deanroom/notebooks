# dotnet 运行时架构

## AOT 编译程序的运行时实例

```mermaid
graph TB
    subgraph "AOT 编译后的进程"
        subgraph "Native Layer"
            NH[Native Host<br/>应用程序]
            RUNTIME[最小运行时<br/>仅包含必要组件]
        end

        subgraph "AOT Runtime Components"
            subgraph "必需组件"
                GC[垃圾回收器]
                THREAD[线程管理]
                EH[异常处理]
            end
            
            subgraph "可选组件"
                style OptionalBox fill:#f0f0f0,stroke-dasharray: 5 5
                DIAG[诊断服务]
                PROF[性能分析]
            end
        end

        subgraph "应用程序代码"
            APP[AOT编译的应用代码]
            LIB[AOT编译的类库]
        end

        %% 连接关系
        NH --> RUNTIME
        RUNTIME --> GC
        RUNTIME --> THREAD
        RUNTIME --> EH
        
        RUNTIME -.-> DIAG
        RUNTIME -.-> PROF
        
        APP --> GC
        APP --> THREAD
        LIB --> GC
    end

    %% 说明性注释
    note1[/"与JIT模式相比：
    1. 没有JIT编译器
    2. 没有程序集加载器
    3. 元数据大幅精简
    4. 运行时组件最小化"/]
    
    style note1 fill:#f9f9f9,stroke:#333,stroke-width:1px
```

## AOT 运行时特点说明：

1. **单一运行时实例**
   - AOT 编译的程序只需要一个最小化的运行时实例
   - 运行时组件被静态链接到应用程序中

2. **必需组件**
   - 垃圾回收器（GC）：内存管理仍然必需
   - 线程管理：处理并发和同步
   - 异常处理：处理运行时异常

3. **精简特性**
   - 没有 JIT 编译器
   - 没有动态程序集加载
   - 元数据信息最小化
   - 类型系统简化

4. **性能优势**
   - 启动更快（无需 JIT）
   - 内存占用更少
   - 可预测的性能表现

5. **限制**
   - 动态特性受限
   - 反射功能受限
   - 运行时不可更新
  