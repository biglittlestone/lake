<h1>Dynamo 扩缩容与可观测性</h1>
<p>代码级剖析 — 基于 dynamo-ascend 仓库 · 2026-09-28</p>

<h2 class="section-title" id="overview">整体架构</h2>

<div class="plain">
Dynamo 是一个大模型推理调度平台。它解决两个核心问题：流量来了怎么动态加机器（扩缩容），出了问题怎么快速定位（可观测性）。整个系统从上到下分四层，每层各管一件事。
</div>

<p>&lt;<SVG_BLOCK_0>&gt;</p>
<h2 class="section-title" id="scaling-intro">扩缩容 — 大白话</h2>

<div class="plain">
扩缩容就是"自动加/减机器"。Dynamo 的难点在于：大模型推理不只是加个 Pod 就完事 — 每个 worker 上有 KV Cache（类似 GPU 上的缓存数据库），加减机器时要把这些缓存数据迁移过去，还不能丢请求。所以扩缩容分三层：K8s 层负责加 Pod，Planner 层决定"要不要加"，KV Cache 层负责"数据怎么搬"。三层联动，缺一不可。
</div>

<h3 id="k8s">1. K8s Operator 层 <span class="badge badge-go">Go</span></h3>

<div class="plain">
想象有一个"总开关"（Adapter CRD），所有想调整副本数的组件（HPA、Planner、运维）都必须通过它来操作。这样做的好处是：防止 HPA 和 Planner 同时改副本数产生冲突。总开关收到指令后，转发给 DGD（Deployment），DGD 再实际调用 K8s API 加/减 Pod。
</div>

<h4>调用链</h4>
<div class="call-chain">
HPA / KEDA / Planner <span class="arrow">→</span> 写 Adapter .spec.replicas
  <span class="arrow">→</span> <span class="fn">ScalingAdapter Reconcile</span> <span style="color:var(--dim)">(scalingadapter_controller.go:59)</span>
  <span class="arrow">→</span> 写 DGD .spec.components[*].replicas
  <span class="arrow">→</span> <span class="fn">DGD Controller Reconcile</span>
  <span class="arrow">→</span> <span class="fn">ScaleResource()</span> <span style="color:var(--dim)">(scale.go:32)</span>
  <span class="arrow">→</span> <span class="fn">kubeClient.SubResource("scale").Update()</span>
  <span class="arrow">→</span> Pod 增减
</div>

<p>&lt;<TOGGLE_BLOCK_0>&gt;</p>
<h4 id="k8s-diag">图解：Pod 扩缩容流程</h4>

<p>&lt;<SVG_BLOCK_1>&gt;</p>
<h3 id="planner">2. Planner 决策层 <span class="badge badge-py">Python</span></h3>

<div class="plain">
Planner 是一个"大脑"，它每 5 秒看一次各个 worker 的性能指标（FPM），然后做两个判断：
<br><br>
<strong>Easy 模式（默认）：</strong> 看 queue 和 KV 利用率，像看水温一样 — 水太热（queue 太长）就加机器，水凉了（queue 很短）就减机器。简单粗暴但有效。
<br><br>
<strong>SLA 模式：</strong> 用线性回归模型预测 — "如果现在加/减一台机器，TTFT（首 token 延迟）会是多少？会不会超过 SLA？" 这是科学决策，需要至少 5 个样本才能开始预测。最关键的是缩容安全校验：缩容前模拟一下"如果把一台机器的活分给其他机器，SLA 还满足吗？"
</div>

<h4>入口调用链</h4>
<div class="call-chain">
<span class="fn">BuiltinLoadPropose.Propose()</span> <span style="color:var(--dim)">(local_planner.py:331)</span>
  <span class="arrow">→</span> <span class="fn">PlannerScalingState.advance_load()</span> <span style="color:var(--dim)">(state_machine.py:218)</span>
  <span class="arrow">→</span> <span class="fn">LoadScalingMixin._advance_load(obs)</span> <span style="color:var(--dim)">(load_scaling.py:48)</span>
  <span class="arrow">→</span> 按 mode 分发: <span class="fn">_advance_load_agg</span> / <span class="fn">_advance_load_disagg</span> / <span class="fn">_advance_load_single</span>
</div>

<h4>两种模式对比</h4>
<table>
<tr><th>模式</th><th>决策方式</th><th>适合场景</th></tr>
<tr><td><strong>Easy 模式</strong> (默认)</td><td>静态阈值：queue/KV util 硬比较</td><td>快速上手，不需要训练数据</td></tr>
<tr><td><strong>SLA 模式</strong></td><td>回归模型预测 TTFT/ITL + 缩容安全校验</td><td>有 SLA 要求，有足够 FPM 数据</td></tr>
</table>

<h4>Easy 模式阈值 <span style="color:var(--dim)">(load_scaling.py:24-35 硬编码)</span></h4>
<table>
<tr><th>信号</th><th>扩容阈值</th><th>缩容阈值</th><th>大白话</th></tr>
<tr><td>Prefill queue / context_length</td><td>≥ 1.0</td><td>&lt; 0.1</td><td>排队超过 context 长度 → 加机器</td></tr>
<tr><td>Decode KV util</td><td>&gt; 100%</td><td>&lt; 60%</td><td>KV 缓存超卖 → 加；空闲太多 → 减</td></tr>
</table>

<h4>SLA 模式核心公式</h4>
<p>模型: <code>wall_time = coef × sum_prefill_tokens + intercept</code>（sklearn LinearRegression）</p>

<p>&lt;<TOGGLE_BLOCK_1>&gt;</p>
<h4 id="planner-diag">图解：缩容安全校验 — 为什么不能随便缩？</h4>

<p>&lt;<SVG_BLOCK_2>&gt;</p>
<h4>默认配置 <span style="color:var(--dim)">(defaults.py)</span></h4>
<table>
<tr><th>参数</th><th>默认值</th><th>说明</th></tr>
<tr><td><code>enable_load_scaling</code></td><td><span class="hl-yellow">False</span></td><td>需手动开启</td></tr>
<tr><td><code>ttft_ms</code></td><td>500</td><td>TTFT SLA 目标</td></tr>
<tr><td><code>itl_ms</code></td><td>50</td><td>ITL SLA 目标</td></tr>
<tr><td><code>load_scaling_down_sensitivity</code></td><td>80</td><td>缩容安全系数 0.8</td></tr>
<tr><td><code>max_throughput_scaling_replicas</code></td><td>8</td><td>最大副本数</td></tr>
<tr><td><code>load_min_observations</code></td><td>5</td><td>回归模型最少样本数</td></tr>
</table>

<h3 id="kv-layer">3. KV Cache 路由层 <span class="badge badge-rust">Rust</span> <span class="badge badge-go">Go</span></h3>

<div class="plain">
KV Cache 是 GPU 上存储的前缀缓存。每个 block 属于某个 worker 所有。当新节点加入时，Hash 环重新划分，原来由旧节点管理的 block 需要迁移到新节点。扩容时新节点"立即可用"（不等迁移完成），缩容时先停止新请求（markDraining），迁移完再删节点。防抖机制确保不会因为一瞬间的流量波动就频繁扩缩容。
</div>

<h4>Router 自动扩缩决策 <span style="color:var(--dim)">(lake/go/router/autoscale.go)</span></h4>
<div class="call-chain">
autoscaleTick() <span style="color:var(--dim)">(line 418)</span>
  <span class="arrow">→</span> <span class="fn">reapDraining()</span> — 清理已完成 drain 的节点
  <span class="arrow">→</span> <span class="fn">flushHotHits()</span> — 上报 KV hit 到控制面
  <span class="arrow">→</span> <span class="fn">sched.LoadSnapshot()</span> — 取调度器快照: QueueLen, InFlight
  <span class="arrow">→</span> <span class="fn">scaler.Evaluate()</span> — 决策
  <span class="arrow">→</span> <span class="fn">applyScale()</span> — 执行
</div>

<h4>防抖动逻辑 <span style="color:var(--dim)">(Evaluate(), line 101)</span></h4>
<div class="plain">
就像一个温度传感器：不是说"超过 4 个请求就扩容"，而是"连续 3 次检查都超过 4，且距离上次操作超过 10 秒"才触发。如果中间恢复正常，计数器立即清零。这防止了"抖动" — 流量一会儿高一会儿低导致反复扩缩容。
</div>

<table>
<tr><th>参数</th><th>默认</th><th>大白话</th></tr>
<tr><td><code>MinNodes</code></td><td>1</td><td>至少 1 个（worker-0 永不移除）</td></tr>
<tr><td><code>MaxNodes</code></td><td>8</td><td>最多 8 个</td></tr>
<tr><td><code>ScaleOutQueueLen</code></td><td>4</td><td>队列 ≥ 4 持续 3 tick → 扩容</td></tr>
<tr><td><code>ScaleInQueueLen</code></td><td>0</td><td>队列为空且 inflight ≤ 1 持续 3 tick → 缩容</td></tr>
<tr><td><code>SustainPeriods</code></td><td>3</td><td>连续 tick 数</td></tr>
<tr><td><code>Cooldown</code></td><td>10s</td><td>两次操作间最少间隔</td></tr>
</table>

<h4>扩容 vs 缩容执行步骤</h4>
<div class="two-col">
  <div>
    <h4 style="color:var(--green)">扩容</h4>
    <div class="call-chain">
<span class="num">1</span>. <span class="fn">nodes.nextID()</span> → <span class="str">"worker-2"</span>
<span class="num">2</span>. <span class="fn">cp.JoinShardNode(RPC)</span> → 迁移列表
<span class="num">3</span>. <span class="fn">nodes.add()</span> → <span class="hl-yellow">立即可路由</span>
<span class="num">4</span>. <span class="fn">syncCapacity()</span> → 总并发 × nodeCount
<span class="num">5</span>. <span class="fn">warmup_plan()</span> → 热数据预热
</div>
  </div>
  <div>
    <h4 style="color:var(--red)">缩容</h4>
    <div class="call-chain">
<span class="num">1</span>. <span class="fn">victim = lastReady()</span> → LIFO 选最新的
<span class="num">2</span>. <span class="fn">markDraining()</span> → 停止新请求
<span class="num">3</span>. <span class="fn">cp.DrainShardNode(RPC)</span> → 迁移计划
  <span class="arrow">失败 → <span class="hl-yellow">回滚</span></span>
<span class="num">4</span>. <span class="fn">reapDraining()</span> → 等待迁移完成
</div>
  </div>
</div>

<h4 id="kv-diag">图解：一致性 Hash 环 — 节点加入时的数据迁移</h4>

<p>&lt;<SVG_BLOCK_3>&gt;</p>
<h3 id="comm-domain">4. 通信域管理 (TP/EP/PP/DP) <span class="badge badge-go">Go</span> <span class="badge badge-py">Python</span></h3>

<div class="plain">
大模型推理不是单个 worker 干活，而是一群 worker 组成了一个"通信域"。比如 TP（张量并行）把一个矩阵拆成几份、每个 GPU 算一份；EP（专家并行）让每个 GPU 负责不同的 MoE 专家。这些 worker 通过 NCCL 集体通信（collective）协同工作。关键问题是：一个 worker 挂了，整个通信域怎么办？
</div>

<h4>并行组的生命周期</h4>
<div class="plain">
TP 和 PP 组在 Pod 创建时就确定了，运行期间不能变更。只有 EP 组支持"弹性伸缩"（elastic EP）— 可以在运行时动态加入/移除 DP worker。Dynamo Operator 本身不创建 torch.distributed 的 process group，它只负责编排 Pod 和基础设施；真正的 NCCL/torch.distributed 组由后端框架（vLLM、SGLang）自己组建。
</div>

<h4>Worker 故障的三种处理路径</h4>

<div class="two-col">
  <div>
    <h4 style="color:var(--red)">① Inter-Pod 故障 — 全组级联删除</h4>
    <div class="call-chain" style="border-left-color:var(--red)">
<span style="color:var(--red)">单个 Pod 进入 Failed 状态</span>
  <span class="arrow">→</span> <span class="fn">FailoverCascadeController</span> <span style="color:var(--dim)">(failover_cascade_controller.go:85)</span>
  <span class="arrow">→</span> 识别 engine group（相同 Grove 标签的所有 Pod）
  <span class="arrow">→</span> <span style="color:var(--red)">全部 force delete（grace=0）</span>
  <span class="arrow">→</span> Grove 重建整个 cohort
  <span class="arrow">→</span> 全新的 NCCL 集体通信
    </div>
    <p style="font-size:.88rem;color:var(--dim);margin-top:.5rem"><strong>原因：</strong> NCCL 集体通信、torch.distributed TCPStore 成员、CUDA IPC 状态无法原地重启 — 部分销毁后会留下半 torn-down 的残留状态。</p>
  </div>
  <div>
    <h4 style="color:var(--green)">② Intra-Pod 故障 — 单 rank 恢复</h4>
    <div class="call-chain" style="border-left-color:var(--green)">
<span style="color:var(--red)">Pod 内 active 容器崩溃</span>
  <span class="arrow">→</span> standby 容器获取 flock 锁
  <span class="arrow">→</span> <span style="color:var(--green)">单 rank 恢复</span>
  <span class="arrow">→</span> 其他 rank 不受影响
  <span class="arrow">→</span> 共享 GPU（DRA ResourceClaims）
    </div>
    <p style="font-size:.88rem;color:var(--dim);margin-top:.5rem"><strong>原因：</strong> active/standby 在同一 Pod 内共享 GPU，standby 已经准备好，只需要接管锁文件。无需 NCCL 重新初始化。</p>
  </div>
</div>

<p>&lt;<TOGGLE_BLOCK_2>&gt;</p>
<h4>图解：Worker 故障恢复路径</h4>

<p>&lt;<SVG_BLOCK_4>&gt;</p>
<h3 id="non-k8s">5. 非 K8s 部署的扩缩容操作</h3>

<div class="plain">
Dynamo 的扩缩容"大脑"（Planner）和 K8s 完全无关 — 它只做一件事：根据实时负载产出 ScalingDecision（目标副本数）。真正把决策变成动作的是"连接器"（PlannerConnector），按部署环境选择不同实现。非 K8s 用的是 VirtualConnector，它通过 etcd 在 Planner 和你的部署脚本之间传递决策 — 像两个人用留言板沟通：Planner 写好目标写在 etcd 上，你的脚本看到后执行，再把"已完成"写回 etcd。
</div>

<h4>5.1 整体架构：PlannerConnector 抽象层</h4>

<table>
<tr><th>连接器</th><th>环境</th><th>执行机制</th><th>核心文件</th></tr>
<tr><td><code>KubernetesConnector</code></td><td>K8s</td><td>DGD Scaling Adapter Scale subresource</td><td><code>connectors/kubernetes.py</code></td></tr>
<tr><td><code>VirtualConnector</code></td><td>非 K8s（裸金属/VM）</td><td>etcd 协调：Coordinator 发决策 → Client 消费执行</td><td><code>connectors/virtual.py</code> + <code>lib/bindings/python/rust/planner.rs</code></td></tr>
<tr><td><code>GlobalPlannerConnector</code></td><td>多集群</td><td>集中式 GlobalPlanner RPC（ScaleRequest/ScaleResponse）</td><td><code>connectors/global_planner.py</code></td></tr>
</table>

<p>选择哪个连接器由环境变量 <code>environment</code> 决定（<code>planner_factory.py:31-68</code>）：</p>
<pre><code><span class="cmt"># planner_factory.py:31-42</span>
<span class="kw">def</span> <span class="fn">construct_connector</span>(config: PlannerConfig, runtime=<span class="kw">None</span>, worker_info_provider=<span class="kw">None</span>) -> PlannerConnector:
    <span class="kw">if</span> config.environment == <span class="str">"global-planner"</span>:
        <span class="kw">return</span> <span class="fn">GlobalPlannerConnector</span>(...)
    <span class="kw">if</span> config.environment == <span class="str">"kubernetes"</span>:
        <span class="kw">return</span> <span class="fn">KubernetesConnector</span>(...)
    <span class="kw">if</span> config.environment == <span class="str">"virtual"</span>:
        <span class="kw">return</span> <span class="fn">VirtualConnector</span>(runtime, config.namespace, worker_info_provider, config.model_name)
    <span class="kw">raise</span> <span class="fn">ValueError</span>(<span class="str">f"Invalid environment: {config.environment}"</span>)
</code></pre>

<h4>5.2 扩缩容策略 — 三种模式</h4>

<div class="plain">
Planner 支持三种扩缩容模式，决定了"怎么算目标副本数"。模式选择通过 <code>mode</code> 配置项（默认 disagg）：
</div>

<table>
<tr><th>模式</th><th>说明</th><th>代码入口</th></tr>
<tr><td><strong>agg</strong>（聚合）</td><td>单一引擎同时做 prefill + decode，合并信号后统一扩缩</td><td><code>_advance_load_agg()</code> (line 296)</td></tr>
<tr><td><strong>disagg</strong>（非聚合，默认）</td><td>Prefill 和 Decode 独立扩缩，各自有独立的决策逻辑</td><td><code>_advance_load_disagg()</code> (line 129)</td></tr>
<tr><td><strong>prefill</strong> / <strong>decode</strong>（单组件）</td><td>只扩缩一个组件类型</td><td><code>_advance_load_single()</code> (line 59)</td></tr>
</table>

<h4>两种决策算法：Easy 模式 vs SLA 模式</h4>

<div class="two-col">
  <div>
    <h4 style="color:var(--green)">Easy 模式（默认）</h4>
    <div class="plain">就像看水温：设几个固定阈值，超过就扩容、低于就缩容。不需要任何历史数据，开箱即用。</div>
    <table>
    <tr><th>信号</th><th>扩容</th><th>缩容</th></tr>
    <tr><td>Prefill queue / context_length</td><td>≥ 1.0</td><td>&lt; 0.1</td></tr>
    <tr><td>Decode KV util</td><td>&gt; 100%</td><td>&lt; 60%</td></tr>
    </table>
    <p style="font-size:.85rem;color:var(--dim)">硬编码在 <code>load_scaling.py:24-35</code>，每次 tick 最多 ±1 副本</p>
  </div>
  <div>
    <h4 style="color:var(--purple)">SLA 模式</h4>
    <div class="plain">用线性回归模型预测"加/减一台机器后 TTFT/ITL 会是多少"，再跟你的 SLA 目标比较。需要至少 5 个 FPM 样本才能开始预测。</div>
    <p>配置:</p>
    <table>
    <tr><th>参数</th><th>默认</th></tr>
    <tr><td><code>ttft_ms</code></td><td>500</td></tr>
    <tr><td><code>itl_ms</code></td><td>50</td></tr>
    <tr><td><code>load_min_observations</code></td><td>5</td></tr>
    </table>
  </div>
</div>

<h4>吞吐量扩缩 vs 负载扩缩</h4>

<div class="plain">
Dynamo 有两套扩缩容算法并行工作：吞吐量扩缩（"慢大脑"，180s 一次，基于 Prometheus 流量预测，提前准备资源）和负载扩缩（"快大脑"，5s 一次，基于实时 FPM 数据，快速响应突发）。当两者同时开启时，吞吐量扩缩只设置一个"下限"（floor），负载扩缩在这个下限之上做微调。
</div>

<table>
<tr><th>维度</th><th>吞吐量扩缩</th><th>负载扩缩</th></tr>
<tr><td><strong>触发源</strong></td><td>Prometheus 流量预测（ARIMA/Kalman）</td><td>实时 FPM（GPU 每次 forward pass）</td></tr>
<tr><td><strong>频率</strong></td><td>180s（慢）</td><td>5s（快）</td></tr>
<tr><td><strong>方向</strong></td><td>预测未来需求，提前准备</td><td>纠正当前 SLA 违规</td></tr>
<tr><td><strong>输入</strong></td><td>预测 RPS, ISL, OSL, KV hit rate</td><td>每引擎 queue tokens, KV util, wall_time</td></tr>
<tr><td><strong>粒度</strong></td><td>单 tick 最多 ±<code>max_throughput_scaling_replicas</code>（默认 8）</td><td>每次 ±1</td></tr>
<tr><td><strong>两者共存时</strong></td><td>设置下限（floor）</td><td>在 floor 之上运行</td></tr>
</table>

<h4>5.3 扩缩容粒度</h4>

<div class="plain">
扩缩容的最小单位是"一个 worker 进程"（即一个 replica），每次 tick 最多增减 1 个。你可以独立控制 prefill 和 decode 的副本数（在 disagg 模式下），但不能单独扩缩某个 DP rank — DP rank 是 worker 内部的事。
</div>

<p>扩缩容目标的结构（<code>defaults.py:138-141</code>）：</p>
<pre><code><span class="kw">class</span> <span class="fn">TargetReplica</span>(BaseModel):
    sub_component_type: SubComponentType   <span class="cmt"># "prefill" 或 "decode"</span>
    component_name: <span class="kw">Optional</span>[str] = <span class="kw">None</span>
    desired_replicas: <span class="kw">int</span>                  <span class="cmt"># 目标副本数</span>
</code></pre>

<h4>5.4 VirtualConnector — 完整的 etcd 协调协议</h4>

<div class="plain">
VirtualConnector 的核心设计是：Planner 侧（Coordinator）和你的部署侧（Client）通过 etcd 共享状态。etcd 的前缀是 <code>v1/{namespace}/planner/</code>，里面有四个键：
</div>

<table>
<tr><th>etcd Key</th><th>写入者</th><th>含义</th></tr>
<tr><td><code>v1/{ns}/planner/num_prefill_workers</code></td><td>Coordinator（Planner 侧）</td><td>期望的 Prefill worker 数</td></tr>
<tr><td><code>v1/{ns}/planner/num_decode_workers</code></td><td>Coordinator（Planner 侧）</td><td>期望的 Decode worker 数</td></tr>
<tr><td><code>v1/{ns}/planner/decision_id</code></td><td>Coordinator（Planner 侧）</td><td>决策版本号，每次递增</td></tr>
<tr><td><code>v1/{ns}/planner/scaled_decision_id</code></td><td>Client（部署侧）</td><td>已完成执行的版本号（确认回执）</td></tr>
</table>

<h4>Coordinator 写决策流程（Rust 实现，planner.rs:99-211）</h4>
<div class="call-chain">
<span class="fn">update_scaling_decision(num_prefill, num_decode)</span>
  <span class="arrow">→</span> 与当前值比较 → 无变化则 skip
  <span class="arrow">→</span> 检查上一轮是否已完成（<span class="fn">is_scaling_ready()</span>）
  <span class="arrow">→</span> 未完成？记录首次跳过时间戳，等 max_wait_time（30min）→ 超时则强制继续
  <span class="arrow">→</span> <span class="hl-yellow">decision_id += 1</span>
  <span class="arrow">→</span> 原子写入 etcd（<span class="fn">kv_put_many</span>，同一 revision）
  <span class="arrow">→</span> 更新本地 decision 缓存
  <span class="arrow">→</span> 重置 skip 时间戳
</div>

<h4>Client 消费决策流程（Rust 实现，planner.rs:397-426）</h4>
<div class="call-chain">
<span class="hl-yellow">VirtualConnectorClient</span>（部署环境使用）
  <span class="arrow">→</span> <span class="fn">wait()</span> — 阻塞等待 etcd watch（<span class="fn">kv_watch_prefix</span>），有新决策才返回
  <span class="arrow">→</span> <span class="fn">get()</span> — 从 etcd prefix read 读取 PlannerDecision(num_prefill, num_decode, decision_id)
  <span class="arrow">→</span> 执行实际的 worker 启停（由用户实现）
  <span class="arrow">→</span> <span class="fn">complete(event)</span> — 写 <code>scaled_decision_id = event.decision_id</code> 到 etcd
</div>

<h4>Coordinator 等待完成（planner.rs:214-263）</h4>
<div class="plain">
Planner 发完决策后调用 <code>wait_for_scaling_completion()</code>，它每 10 秒轮询一次 etcd 中的 <code>scaled_decision_id</code>，最多等 30 分钟（180 次）。当 <code>scaled_decision_id ≥ decision_id</code> 时认为完成。如果超时，Planner 继续下一轮 tick（不会卡死）。
</div>

<h4>5.5 完整示例代码 — 非 K8s 扩缩容</h4>

<div class="plain">
下面是一段完整的测试代码，展示了 Planner 侧（VirtualConnector）和部署侧（VirtualConnectorClient）如何协作。这就是你实际部署时需要写的逻辑。
</div>

<pre><code><span class="cmt"># ============================================</span>
<span class="cmt"># 来源: test_virtual_connector.py（真实测试用例）</span>
<span class="cmt"># ============================================</span>

<span class="cmt"># --- 前置条件：启动 etcd 和 NATS ---</span>
<span class="cmt"># docker compose -f dev/docker-compose.yml up -d</span>

<span class="cmt"># --- PLANNER 侧：创建 VirtualConnector ---</span>
<span class="kw">from</span> dynamo._core <span class="kw">import</span> DistributedRuntime, VirtualConnectorClient
<span class="kw">from</span> dynamo.planner <span class="kw">import</span> SubComponentType, TargetReplica, VirtualConnector

runtime = <span class="fn">DistributedRuntime</span>(loop, <span class="str">"etcd"</span>, <span class="str">"nats"</span>)

c = <span class="fn">VirtualConnector</span>(
    runtime,
    <span class="str">"my_deployment"</span>,              <span class="cmt"># namespace，Planner 和 Client 必须相同</span>
    worker_info_provider=<span class="fn">DefaultWorkerInfoProvider</span>(),
    model_name=<span class="str">"sglang"</span>,
)
<span class="kw">await</span> c.<span class="fn">async_init</span>()

<span class="cmt"># --- 第一次扩缩：prefill=1, decode=2 ---</span>
replicas = [
    <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.PREFILL, desired_replicas=<span class="num">1</span>),
    <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.DECODE, desired_replicas=<span class="num">2</span>),
]
<span class="kw">await</span> c.<span class="fn">set_component_replicas</span>(replicas, blocking=<span class="kw">False</span>)

<span class="cmt"># --- CLIENT 侧：部署环境消费决策 ---</span>
client = <span class="fn">VirtualConnectorClient</span>(runtime, <span class="str">"my_deployment"</span>)

event = <span class="kw">await</span> client.<span class="fn">get</span>()
<span class="cmt"># 读取到: num_prefill_workers=1, num_decode_workers=2, decision_id=0</span>
<span class="kw">assert</span> event.num_prefill_workers == <span class="num">1</span>
<span class="kw">assert</span> event.num_decode_workers == <span class="num">2</span>

<span class="cmt"># 在这里执行实际的扩缩容操作：</span>
<span class="cmt"># - 启动/停止 worker 进程</span>
<span class="cmt"># - 更新负载均衡器配置</span>
<span class="cmt"># - 等等（具体实现由用户自己完成）</span>

<span class="cmt"># 完成后回写确认</span>
<span class="kw">await</span> client.<span class="fn">complete</span>(event)

<span class="cmt"># Planner 侧等待完成</span>
<span class="kw">await</span> c._<span class="fn">wait_for_scaling_completion</span>()

<span class="cmt"># --- 第二次扩缩：使用 wait() 监听新决策 ---</span>
<span class="kw">async def</span> <span class="fn">next_scaling_decision</span>(c):
    replicas = [
        <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.PREFILL, desired_replicas=<span class="num">5</span>),
        <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.DECODE, desired_replicas=<span class="num">8</span>),
    ]
    <span class="kw">await</span> c.<span class="fn">set_component_replicas</span>(replicas, blocking=<span class="kw">False</span>)

task = asyncio.<span class="fn">create_task</span>(<span class="fn">next_scaling_decision</span>(c))
<span class="kw">await</span> client.<span class="fn">wait</span>()        <span class="cmt"># 阻塞直到 etcd 有新决策</span>
<span class="kw">await</span> task

event = <span class="kw">await</span> client.<span class="fn">get</span>()
<span class="kw">assert</span> event.num_prefill_workers == <span class="num">5</span>
<span class="kw">assert</span> event.num_decode_workers == <span class="num">8</span>
<span class="kw">assert</span> event.decision_id == <span class="num">1</span>  <span class="cmt"># decision_id 递增</span>
<span class="kw">await</span> client.<span class="fn">complete</span>(event)
<span class="kw">await</span> c._<span class="fn">wait_for_scaling_completion</span>()

<span class="cmt"># --- 缩容到零 ---</span>
replicas = [
    <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.PREFILL, desired_replicas=<span class="num">0</span>),
    <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.DECODE, desired_replicas=<span class="num">0</span>),
]
<span class="kw">await</span> c.<span class="fn">set_component_replicas</span>(replicas, blocking=<span class="kw">False</span>)
</code></pre>

<h4>5.6 便捷的扩缩接口</h4>

<div class="plain">
除了直接设置目标副本数，VirtualConnector 还提供了三个便捷方法 — <code>add_component</code>（+1）、<code>remove_component</code>（-1）、<code>set_component_replicas</code>（精确设置）。底层都是通过 <code>update_scaling_decision</code> 写入 etcd。
</div>

<pre><code><span class="cmt"># 增加一个 Prefill worker</span>
<span class="kw">await</span> c.<span class="fn">add_component</span>(SubComponentType.PREFILL, blocking=<span class="kw">True</span>)

<span class="cmt"># 减少一个 Decode worker（最少减到 0）</span>
<span class="kw">await</span> c.<span class="fn">remove_component</span>(SubComponentType.DECODE, blocking=<span class="kw">True</span>)

<span class="cmt"># 精确设置</span>
<span class="kw">await</span> c.<span class="fn">set_component_replicas</span>([
    <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.PREFILL, desired_replicas=<span class="num">3</span>),
    <span class="fn">TargetReplica</span>(sub_component_type=SubComponentType.DECODE, desired_replicas=<span class="num">6</span>),
], blocking=<span class="kw">True</span>)
</code></pre>

<h4>5.7 Worker 发现机制（无 K8s）</h4>

<div class="plain">
非 K8s 环境下，"当前有多少 worker 在运行"不查 K8s API，而是通过 Dynamo runtime 的端点发现。每个 worker 启动时注册自己的端点到 etcd+NATS，Planner 通过查询端点实例 ID 来统计实际数量。
</div>

<div class="call-chain">
<span class="fn">get_actual_worker_counts()</span> <span style="color:var(--dim)">(virtual.py:194-220)</span>
  <span class="arrow">→</span> <span class="fn">_get_actual_worker_count(PREFILL)</span> <span style="color:var(--dim)">(virtual.py:222-249)</span>
  <span class="arrow">→</span> 构建 endpoint 名: <span class="str">"namespace.component.endpoint"</span>
  <span class="arrow">→</span> <span class="fn">runtime.endpoint(name).client()</span> 获取运行时客户端
  <span class="arrow">→</span> <span class="fn">client.instance_ids()</span> → 返回所有注册的实例
  <span class="arrow">→</span> <span class="fn">len(instance_ids)</span> = 实际 worker 数量
</div>

<p>稳定性判断：不仅需要 etcd 回执确认，还需要"发现的实际数量 == 期望数量"（virtual.py:216-219）。</p>

<h4>5.8 Planner 配置</h4>

<div class="plain">
Planner 的行为由 JSON/YAML 配置文件控制（非环境变量）。核心是 <code>environment</code>（选择连接器）和 <code>mode</code>（选择扩缩策略）。注意：<code>optimization_target</code> 的值会自动覆盖 <code>enable_load_scaling</code> 和 <code>enable_throughput_scaling</code> 的默认值——Easy 模式（非 sla）会强制开启 load scaling、关闭 throughput scaling。
</div>

<h5>5.8.1 核心配置项</h5>
<table>
<tr><th>配置项</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>environment</code></td><td><code>kubernetes</code></td><td>改为 <code>virtual</code> 启用非 K8s 模式</td><td>"在哪部署"</td></tr>
<tr><td><code>mode</code></td><td><code>disagg</code></td><td><code>agg</code> / <code>disagg</code> / <code>prefill</code> / <code>decode</code></td><td>"怎么扩"</td></tr>
<tr><td><code>optimization_target</code></td><td><code>throughput</code></td><td><code>throughput</code>/latency/load(Easy) / <code>sla</code>(回归模型)</td><td>"扩缩依据"</td></tr>
<tr><td><code>enable_load_scaling</code></td><td><code>False</code> → 被验证器覆盖</td><td>Easy 模式强制 <code>True</code>，SLA 模式默认 <code>False</code></td><td>"实时负载扩缩"</td></tr>
<tr><td><code>enable_throughput_scaling</code></td><td><code>True</code> → 被验证器覆盖</td><td>Easy 模式强制 <code>False</code>，SLA 模式默认 <code>True</code></td><td>"流量预测扩缩"</td></tr>
<tr><td><code>advisory</code></td><td><code>False</code></td><td>设为 <code>True</code> 为"只算不执行"（dry-run）</td><td>"模拟模式"</td></tr>
<tr><td><code>load_adjustment_interval_seconds</code></td><td>5</td><td>负载扩缩频率</td><td>"多久看一次负载"</td></tr>
<tr><td><code>throughput_adjustment_interval_seconds</code></td><td>180</td><td>吞吐量扩缩频率</td><td>"多久预测一次流量"</td></tr>
<tr><td><code>max_gpu_budget</code></td><td>8</td><td>GPU 预算上限</td><td>"最多用多少 GPU"</td></tr>
<tr><td><code>min_endpoint</code></td><td>1</td><td>最少副本数</td><td>"最少保留几个"</td></tr>
</table>

<h5>5.8.2 环境变量（Planner 进程）</h5>
<div class="plain">
Planner 本身不是通过命令行参数配置，而是读取 JSON/YAML 配置文件。但以下环境变量会影响其行为：
</div>

<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th></tr>
<tr><td><code>DYN_NAMESPACE</code></td><td><code>dynamo</code></td><td>命名空间，用于模型发现和端点注册</td></tr>
<tr><td><code>DYN_PARENT_DGD_K8S_NAME</code></td><td>无（K8s 模式必填）</td><td>K8s 中父 DGD 名称，未设置会报错</td></tr>
<tr><td><code>PROMETHEUS_ENDPOINT</code></td><td>K8s 内部地址</td><td>Prometheus 服务器 URL，Planner 用来查询流量数据</td></tr>
<tr><td><code>PROMETHEUS_TOKEN</code></td><td>无</td><td>Prometheus 认证 Token</td></tr>
<tr><td><code>PLANNER_PROMETHEUS_PORT</code></td><td><code>0</code>（关闭）</td><td>Planner 自身的指标暴露端口</td></tr>
<tr><td><code>SCALING_CHECK_INTERVAL</code></td><td>10</td><td>Virtual connector 轮询间隔（秒）</td></tr>
<tr><td><code>SCALING_MAX_WAIT_TIME</code></td><td>1800</td><td>Virtual connector 最大等待时间（秒）</td></tr>
<tr><td><code>ETCD_ENDPOINTS</code></td><td><code>http://localhost:2379</code></td><td>etcd 地址（Virtual connector 用）</td></tr>
</table>

<h5>5.8.3 扩缩模式自动覆盖规则</h5>
<div class="plain">
Planner 配置文件中的 <code>enable_load_scaling</code> 和 <code>enable_throughput_scaling</code> 默认值会被验证器自动覆盖。这是代码层面的强制逻辑（planner_config.py:962-986）：
</div>

<table>
<tr><th>optimization_target</th><th>enable_load_scaling 最终值</th><th>enable_throughput_scaling 最终值</th><th>大白话</th></tr>
<tr><td><code>throughput</code>（默认）</td><td><code>True</code>（强制）</td><td><code>False</code>（强制）</td><td>"Easy 模式：只看实时负载"</td></tr>
<tr><td><code>latency</code></td><td><code>True</code>（强制）</td><td><code>False</code>（强制）</td><td>"Easy 模式：只看延迟阈值"</td></tr>
<tr><td><code>load</code></td><td><code>True</code>（强制）</td><td><code>False</code>（强制）</td><td>"Easy 模式：只看 FPM 负载"</td></tr>
<tr><td><code>sla</code></td><td>尊重配置值（默认 False）</td><td>尊重配置值（默认 True）</td><td>"SLA 模式：用回归模型预测"</td></tr>
</table>

<div class="plain">
两种模式可以同时开启（仅 SLA 模式下可能），此时 throughput scaling 产出"下限"，load scaling 在此之上运行。至少必须开启一个，否则验证器直接报错。
</div>

<h5>5.8.4 扩缩决策跳过条件</h5>
<div class="plain">
Planner 并非每个 tick 都会产出扩缩决策。以下情况会直接跳过（代码中的 early-return 路径）：
</div>

<table>
<tr><th>跳过条件</th><th>触发场景</th><th>大白话</th></tr>
<tr><td><code>model_not_ready</code></td><td>性能模型无法产出容量预估</td><td>"还没摸清性能，不扩不缩"</td></tr>
<tr><td><code>scaling_in_progress</code></td><td>上次扩缩操作还没完成</td><td>"上一轮还没落地，等一等"</td></tr>
<tr><td><code>no_fpm_data</code></td><td>没有收到任何 FPM 数据</td><td>"没数据，瞎猜不如不猜"</td></tr>
<tr><td><code>worker_count_mismatch</code></td><td>FPM worker 数 ≠ DGD 配置数</td><td>"对不上数，先等等"</td></tr>
<tr><td><code>insufficient_data</code></td><td>观测数据还不够多</td><td>"数据太少，不敢决策"</td></tr>
<tr><td><code>scale_down_refused_consolidation</code></td><td>缩容后预测违反 SLA 安全边际</td><td>"缩了会超 SLA，不缩"</td></tr>
<tr><td><code>advisory=True</code></td><td>模拟模式</td><td>"只算不做"</td></tr>
<tr><td><code>already_at_desired_count</code></td><td>当前数量 = 目标数量</td><td>"已经在目标了，不用动"</td></tr>
</table>

<h4>5.9 裸金属部署完整流程</h4>

<div class="plain">
裸金属部署分两步：先启动基础设施（etcd + NATS），再启动业务组件。扩缩容时，Planner 通过 VirtualConnector 产出决策，你的部署脚本通过 VirtualConnectorClient 消费决策并执行。项目自带的启动脚本可以参考，但生产环境通常需要自己写更健壮的进程管理脚本。
</div>

<table>
<tr><th>步骤</th><th>命令/脚本</th><th>说明</th></tr>
<tr><td>① 启动基础设施</td><td><code>docker compose -f dev/docker-compose.yml up -d</code></td><td>启动 etcd + NATS</td></tr>
<tr><td>② 启动 workers</td><td><code>python -m dynamo.vllm --model $MODEL &amp;</code></td><td>每个 worker 一个进程，设置不同的 DYN_SYSTEM_PORT</td></tr>
<tr><td>③ 启动前端路由</td><td><code>python -m dynamo.frontend --router-mode kv &amp;</code></td><td>自动发现注册的 workers</td></tr>
<tr><td>④ 启动 Planner</td><td><code>environment=virtual python -m dynamo.planner &amp;</code></td><td>通过 VirtualConnector 连接 etcd</td></tr>
<tr><td>⑤ 部署脚本</td><td>自写脚本监听 VirtualConnectorClient</td><td>消费决策 → 启停 worker 进程</td></tr>
</table>

<pre><code><span class="cmt"># 参考: examples/backends/vllm/launch/agg_router.sh</span>

<span class="cmt"># 启动前端路由 + KV router</span>
python -m dynamo.frontend --router-mode kv --http-port <span class="num">8000</span> <span class="op">&amp;</span>

<span class="cmt"># 启动 worker（每个 GPU 一个）</span>
<span class="cmt"># 关键: 必须设 DYN_SYSTEM_PORT 才会有 /metrics；设 DYN_FPM_TRACE 才会发 FPM 数据</span>
CUDA_VISIBLE_DEVICES=<span class="num">0</span> DYN_SYSTEM_PORT=<span class="num">8081</span> DYN_FPM_TRACE=<span class="num">1</span> \
  python3 -m dynamo.vllm --model $MODEL <span class="op">&amp;</span>
CUDA_VISIBLE_DEVICES=<span class="num">1</span> DYN_SYSTEM_PORT=<span class="num">8082</span> DYN_FPM_TRACE=<span class="num">1</span> \
  python3 -m dynamo.vllm --model $MODEL <span class="op">&amp;</span>

<span class="cmt"># 启动 Planner（非 K8s 模式）</span>
environment=<span class="str">virtual</span> \
enable_load_scaling=<span class="str">True</span> \
mode=<span class="str">disagg</span> \
python3 -m dynamo.planner <span class="op">&amp;</span></code></pre>

<h4>5.10 Lake Router 自动扩缩（独立于 Planner）</h4>

<div class="plain">
除了 Planner，Lake 路由器还自带了一个轻量级的自动扩缩器。它不依赖 Planner，只看路由器的"队列深度"来决定加不减节点。这个 autoscaler 默认关闭，设置 <code>LAKE_AUTOSCALE=1</code> 开启。扩容时通过 CP RPC 通知控制面加入新节点，缩容时先标记 draining 再移除。
</div>

<div class="call-chain">
<span class="hl-yellow">LAKE_AUTOSCALE=1</span>
  <span class="arrow">→</span> <span class="fn">autoscaleTick()</span> 每 2s <span style="color:var(--dim)">(server.go:145)</span>
  <span class="arrow">→</span> <span class="fn">reapDraining()</span> — 清理已 drain 完的节点
  <span class="arrow">→</span> <span class="fn">scaler.Evaluate()</span> — 带防抖的决策
  <span class="arrow">→</span> <span class="fn">applyScale()</span>
</div>

<p>&lt;<TOGGLE_BLOCK_3>&gt;</p>
<h2 class="section-title" id="obs-intro">可观测性 — 大白话</h2>

<div class="plain">
可观测性就是"怎么知道系统在干什么、有没有问题"。Dynamo 用三个手段：
<br><br>
<strong>Metrics（指标）</strong> — 就像汽车仪表盘：实时看到请求数、延迟、缓存命中率。Prometheus 定时来"看"（Pull）。
<br><br>
<strong>Tracing（链路追踪）</strong> — 追踪一个请求从进到出的完整路径，每个环节耗时多少。通过 W3C traceparent header 跨进程传播。
<br><br>
<strong>FPM（性能指标）</strong> — 从 vLLM 引擎每次 forward pass 后采集的真实性能数据，直接驱动扩缩容决策。这是扩缩容的"眼睛"。
</div>

<h3 id="metrics">1. Metrics 指标 <span class="badge badge-rust">Rust</span></h3>

<div class="plain">
Dynamo 的指标系统有一个四层层次结构：DRT → Namespace → Component → Endpoint。你每创建一个 metric，系统会自动加上 namespace、component、endpoint、worker_id 四个标签，你不需要手动写。Prometheus 来抓取 /metrics 时，系统先运行回调函数更新 gauge 值，然后把所有子注册表的指标合并去重，输出 Prometheus 文本格式。注意：整个系统 server（/metrics、/health、/live）默认关闭，需设置 <code>DYN_SYSTEM_PORT >= 0</code> 才会开启。
</div>

<h4>四层层次结构 + 自动标签</h4>
<div class="call-chain">
<span class="hl-yellow">DRT</span> (DistributedRuntime)
  <span class="arrow">└──</span> <span class="hl-yellow">Namespace</span>     → 注入 <span class="str">dynamo_namespace</span>
       <span class="arrow">└──</span> <span class="hl-yellow">Component</span>     → 注入 <span class="str">dynamo_component</span>
            <span class="arrow">└──</span> <span class="hl-yellow">Endpoint</span>      → 注入 <span class="str">dynamo_endpoint</span> + <span class="str">worker_id</span>
</div>

<h4 id="metrics-diag">图解：/metrics 抓取流程</h4>

<p>&lt;<SVG_BLOCK_5>&gt;</p>
<h4>关键 Metric 前缀 <span style="color:var(--dim)">(prometheus_names.rs, 1238 行)</span></h4>
<table>
<tr><th>前缀</th><th>核心指标</th><th>大白话</th></tr>
<tr><td><code>dynamo_frontend_</code></td><td>requests_total, ttft, itl, kv_hit_rate</td><td>前端：请求数、延迟、命中率</td></tr>
<tr><td><code>dynamo_router_</code></td><td>router_ttft, router_kv_hit_rate</td><td>路由：路由维度延迟和命中率</td></tr>
<tr><td><code>dynamo_tokio_</code></td><td>worker_busy_ratio, global_queue_depth</td><td>运行时：Tokio worker 忙度、队列深度</td></tr>
<tr><td><code>dynamo_routing_overhead_</code></td><td>block_hashing_ms, scheduling_ms</td><td>路由开销：各阶段耗时</td></tr>
<tr><td><code>dynamo_planner_</code></td><td>num_prefill_replicas, observed_ttft_ms</td><td>Planner：当前副本数、观测到的延迟</td></tr>
</table>

<h3 id="prometheus-setup">1.5 如何接入 Prometheus — 从零到一的完整步骤 <span class="badge badge-rust">Rust</span> <span class="badge badge-py">Python</span></h3>

<div class="plain">
接入 Prometheus 的"大白话"版：Dynamo 每个组件启动时都会自带一个 /metrics HTTP 端点，就像每家店门口摆了个"营业数据牌"。Prometheus 是个"巡逻员"，定时来各家店门口看一眼数据牌，把数据抄下来存到自己的数据库里。Grafana 是个"看板"，从 Prometheus 读取数据画成图表。你只需要做三件事：① 开启并确认 Dynamo 的 /metrics 端口能访问；② 告诉 Prometheus 去哪里看数据牌；③ 在 Grafana 里导入仪表板。
</div>

<div class="plain" style="border-color: rgba(248,81,73,.4); background: linear-gradient(135deg, rgba(248,81,73,.08), rgba(210,153,34,.08));">
⚠️ <strong>关键前提：/metrics 默认是关闭的！</strong><br>
<code>DYN_SYSTEM_PORT</code> 的默认值是 <code>-1</code>（禁用），所以直接 <code>python -m dynamo.vllm</code> 启动时 <strong>不会</strong> 启动 system status server，也就没有 <code>/metrics</code>、<code>/health</code>、<code>/live</code>。<br>
必须设置 <code>DYN_SYSTEM_PORT=&lt;端口号&gt;</code>（>=0）才会开启。例如 <code>DYN_SYSTEM_PORT=8081</code>。
</div>

<h4>第一步：开启 /metrics 端点</h4>
<div class="plain">
每个 Dynamo 组件（frontend、backend worker、router）启动时，如果设置了 <code>DYN_SYSTEM_PORT >= 0</code>，就会自动启动一个 axum HTTP 服务器，暴露三个端点：<code>/metrics</code>（指标数据）、<code>/health</code>（健康检查）、<code>/live</code>（存活检查）。worker 之间端口不冲突，因为每个 worker 设了不同的 DYN_SYSTEM_PORT。
</div>

<pre><code><span class="cmt"># ✅ 正确：设置 DYN_SYSTEM_PORT 才会开启 /metrics</span>
DYN_SYSTEM_PORT=<span class="num">8081</span> python -m dynamo.vllm --model $MODEL

<span class="cmt"># ❌ 错误：不设置 DYN_SYSTEM_PORT，/metrics 不可用</span>
python -m dynamo.vllm --model $MODEL

<span class="cmt"># Headless 模式例外：即使设了 DYN_SYSTEM_PORT 也不开启（绕过 DistributedRuntime）</span>
DYN_SYSTEM_PORT=<span class="num">8081</span> python -m dynamo.vllm --model $MODEL --headless  <span class="cmt"># ❌ /metrics 不可用</span></code></pre>

<h5>端点启动链</h5>
<div class="call-chain">
<span class="hl-yellow">system_status_server.rs</span> <span style="color:var(--dim)">(axum HTTP server 初始化)</span>
  <span class="arrow">→</span> 创建 TcpListener 绑定 <code>0.0.0.0:{DYN_SYSTEM_PORT}</code>
  <span class="arrow">→</span> 注册路由: <code>GET /metrics</code>, <code>GET /health</code>, <code>GET /live</code>
  <span class="arrow">→</span> 后台 tokio task 持续监听
</div>

<h5>/metrics 请求处理链</h5>
<div class="call-chain">
<code>GET /metrics</code> → <span class="fn">metrics_handler(state)</span> <span style="color:var(--dim)">(system_status_server.rs:336)</span>
  <span class="arrow">→</span> <span class="fn">state.drt().metrics()</span> 获取 MetricsRegistry 根注册表
  <span class="arrow">→</span> <span class="fn">prometheus_expfmt()</span> <span style="color:var(--dim)">(metrics.rs)</span>
  <span class="arrow">→</span> ① 收集所有 child registry（Namespace → Component → Endpoint 层次树）
  <span class="arrow">→</span> ② 运行 UpdateCallback（实时计算 gauge 值，如当前 inflight 请求数）
  <span class="arrow">→</span> ③ FamilyMerger 去重合并（按 "name|key=value" 构建唯一键，防止重复）
  <span class="arrow">→</span> ④ prometheus-text-encode 编码为 Prometheus 标准文本格式
  <span class="arrow">→</span> ⑤ 追加 ExpositionFormatCallback 自定义文本
  <span class="arrow">→</span> HTTP 200 + <code>text/plain; charset=utf-8</code>
</div>

<h5>验证端点</h5>
<pre><code><span class="cmt"># 验证 frontend 的 /metrics 端点</span>
curl http://localhost:8000/metrics | head -20

<span class="cmt"># 验证 backend worker 的 /metrics 端点</span>
curl http://localhost:8081/metrics | head -20

<span class="cmt"># 预期输出示例:</span>
# HELP dynamo_frontend_requests_total Total HTTP requests
# TYPE dynamo_frontend_requests_total counter
dynamo_frontend_requests_total{model="Qwen2.5-7B",worker_id="frontend-0",finish_reason="stop"} 1523
dynamo_frontend_requests_total{model="Qwen2.5-7B",worker_id="frontend-0",finish_reason="length"} 47

# HELP dynamo_frontend_time_to_first_token_seconds Time to first token
# TYPE dynamo_frontend_time_to_first_token_seconds histogram
dynamo_frontend_time_to_first_token_seconds_bucket{le="0.1"} 0
dynamo_frontend_time_to_first_token_seconds_bucket{le="0.25"} 120
...</code></pre>

<h4>第二步：配置 Prometheus 抓取</h4>
<div class="plain">
Prometheus 需要一个配置文件来知道"去哪里拉数据、多久拉一次"。Dynamo 项目自带了一份开箱即用的配置文件，你只需要修改 targets 地址为你实际的组件地址。
</div>

<h5>2A. 开发环境 — 静态配置</h5>
<div class="plain">
开发时每个组件的地址固定，直接写死即可。如果你在 Docker 里运行 Prometheus，用 <code>host.docker.internal</code> 可以访问宿主机的端口。
</div>

<p>&lt;<TOGGLE_BLOCK_4>&gt;</p>
<h5>2B. 生产环境 K8s — ServiceMonitor</h5>
<div class="plain">
在 K8s 上不需要手动写每个 Pod 的地址 — 用 Prometheus Operator 的 ServiceMonitor，Prometheus 会自动发现所有带标签的 Pod。原理是：ServiceMonitor 通过 label selector 匹配 Service，Service 匹配 Pod，Prometheus Operator 自动把这些 Pod 的 IP:Port 生成 scrape 配置。
</div>

<pre><code><span class="cmt"># 生产环境使用 Prometheus Operator ServiceMonitor</span>
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dynamo-monitor
  labels:
    release: prometheus    <span class="cmt"># 必须与 Prometheus Operator 的 release name 匹配</span>
spec:
  selector:
    matchLabels:
      app: dynamo
  endpoints:
    - port: metrics       <span class="cmt"># 对应 Pod 的 DYN_SYSTEM_PORT</span>
      interval: 10s
      path: /metrics
  namespaceSelector:
    matchNames: [dynamo]

<span class="cmt"># 对应的 Service 需要这样定义:</span>
apiVersion: v1
kind: Service
metadata:
  name: dynamo-backend-svc
  labels:
    app: dynamo
spec:
  ports:
    - name: metrics
      port: 8081
      targetPort: 8081
  selector:
    app: dynamo-backend</code></pre>

<h5>2C. 裸金属/虚拟机 — 服务发现脚本</h5>
<div class="plain">
如果不是 K8s 也不想手动写配置文件，可以用"文件服务发现"：写一个脚本定期生成 JSON 文件列出所有 target，Prometheus 会自动读取。这样新增/移除 worker 时只需更新文件。
</div>

<pre><code><span class="cmt"># prometheus.yml - 文件服务发现</span>
scrape_configs:
  - job_name: <span class="str">"dynamo-backend"</span>
    file_sd_configs:
      - files:
          - <span class="str">"/etc/prometheus/dynamo-targets.json"</span>
      refresh_interval: 30s

<span class="cmt"># /etc/prometheus/dynamo-targets.json - 由脚本定期生成</span>
[
  {
    "targets": ["192.168.1.10:8081", "192.168.1.11:8081"],
    "labels": {"worker_type": "prefill", "env": "production"}
  },
  {
    "targets": ["192.168.1.12:8081", "192.168.1.13:8081"],
    "labels": {"worker_type": "decode", "env": "production"}
  }
]

<span class="cmt"># 生成脚本示例 (bash)</span>
<span class="kw">#!/bin/bash</span>
<span class="cmt"># 通过 etcd 发现所有注册的 worker</span>
ETCD_ENDPOINTS=<span class="str">"http://localhost:2379"</span>
PREFIX=<span class="str">"v1/default/planner/"</span>
targets=$(etcdctl --endpoints=$ETCD_ENDPOINTS get $PREFIX --prefix --print-value-only)
<span class="cmt"># 解析为 JSON 写入 /etc/prometheus/dynamo-targets.json</span></code></pre>

<h4>第三步：启动可观测性栈</h4>

<h5>3A. 一键启动（推荐）</h5>
<div class="plain">
Dynamo 提供了一键启动完整可观测性栈的 Docker Compose 文件。一条命令就能启动 Prometheus + Grafana + Tempo + Loki + OTel Collector + DCGM Exporter + NATS Exporter，所有组件预配置好数据源连接。
</div>

<pre><code><span class="cmt"># 进入 dynamo/dev/observability/ 目录</span>
<span class="kw">cd</span> dynamo/dev/observability/

<span class="cmt"># 一键启动</span>
docker compose -f docker-observability.yml up -d

<span class="cmt"># 查看各服务状态</span>
docker compose ps

<span class="cmt"># 预期输出:</span>
<span class="cmt"># NAME                          STATUS</span>
<span class="cmt"># observability-prometheus-1    Up (healthy)</span>
<span class="cmt"># observability-grafana-1       Up (healthy)</span>
<span class="cmt"># observability-tempo-1         Up</span>
<span class="cmt"># observability-loki-1          Up</span>
<span class="cmt"># observability-otel-collector-1 Up</span>
<span class="cmt"># observability-dcgm-exporter-1 Up</span>
<span class="cmt"># observability-nats-exporter-1 Up</span></code></pre>

<h5>3B. 可观测性栈各组件说明</h5>
<table>
<tr><th>组件</th><th>端口</th><th>作用</th><th>大白话</th><th>访问地址</th></tr>
<tr><td>Prometheus</td><td>9090</td><td>时序指标数据库，定时拉取 /metrics</td><td>"数据采集员" — 定时来抄表</td><td>http://localhost:9090</td></tr>
<tr><td>Grafana</td><td>3000</td><td>可视化仪表板，展示 Prometheus/Loki/Tempo 数据</td><td>"看板" — 把数据画成图</td><td>http://localhost:3000 (admin/admin)</td></tr>
<tr><td>Tempo</td><td>3200</td><td>分布式链路追踪存储，接收 OTLP 数据</td><td>"追踪数据库" — 记录请求全链路</td><td>Grafana → Explore → Tempo</td></tr>
<tr><td>Loki</td><td>3100</td><td>日志聚合，通过 label 索引日志</td><td>"日志仓库" — 收集所有日志</td><td>Grafana → Explore → Loki</td></tr>
<tr><td>OTel Collector</td><td>4317/4318</td><td>接收 OTLP 格式 metrics/traces/logs，转发到对应后端</td><td>"数据中转站" — 收数据分发给各系统</td><td>无需直接访问</td></tr>
<tr><td>DCGM Exporter</td><td>9401</td><td>NVIDIA DCGM 的 Prometheus 指标导出器</td><td>"GPU 监控探头" — 采集 GPU 硬件指标</td><td>Prometheus 自动抓取</td></tr>
<tr><td>NATS Exporter</td><td>7777</td><td>NATS 消息中间件的 Prometheus 指标</td><td>"消息队列探头" — 监控 NATS 健康</td><td>Prometheus 自动抓取</td></tr>
</table>

<h5>3C. 最小启动（只接 Prometheus）</h5>
<div class="plain">
如果你只想看指标、暂时不关心链路追踪和日志，可以只启动 Prometheus 和 Grafana。
</div>

<pre><code><span class="cmt"># 只启动 Prometheus</span>
docker run -d --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/dynamo/dev/observability/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:latest

<span class="cmt"># 只启动 Grafana</span>
docker run -d --name grafana \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana:latest

<span class="cmt"># 验证 Prometheus 是否抓取成功</span>
curl http://localhost:9090/api/v1/targets | python3 -m json.tool

<span class="cmt"># 预期: states 包含 UP 的 target</span></code></pre>

<h5>3D. Grafana 配置数据源</h5>
<div class="plain">
Grafana 启动后需要配置数据源。如果使用 docker-observability.yml 一键启动，数据源已经预配置好了。如果手动启动，需要在 Grafana UI 中手动添加。
</div>

<table>
<tr><th>步骤</th><th>操作</th><th>配置值</th></tr>
<tr><td>①</td><td>打开 Grafana → Configuration → Data Sources → Add data source</td><td>—</td></tr>
<tr><td>②</td><td>选择 Prometheus</td><td>—</td></tr>
<tr><td>③</td><td>URL 设置为</td><td><code>http://prometheus:9090</code>（Docker 网络内）或 <code>http://localhost:9090</code>（宿主机）</td></tr>
<tr><td>④</td><td>点击 "Save &amp; Test"</td><td>应显示 "Data source is working"</td></tr>
<tr><td>⑤</td><td>如需 Tempo：选择 Tempo → URL <code>http://tempo:3200</code></td><td>—</td></tr>
<tr><td>⑥</td><td>如需 Loki：选择 Loki → URL <code>http://loki:3100</code></td><td>—</td></tr>
</table>

<h5>3E. 导入 Grafana 仪表板</h5>
<div class="plain">
Dynamo 自带了多个 Grafana 仪表板 JSON 文件，位于 <code>dynamo/dev/observability/grafana_dashboards/</code>。导入方式：Grafana → Dashboards → Import → Upload JSON file。如果使用了 docker-observability.yml 一键启动，仪表板通常会自动加载。
</div>

<table>
<tr><th>仪表板</th><th>文件</th><th>内容</th><th>大白话</th></tr>
<tr><td>主仪表板</td><td><code>dynamo.json</code></td><td>请求数、TTFT/ITL、KV 命中率、队列深度</td><td>"总览看板" — 系统整体健康一目了然</td></tr>
<tr><td>K8s Operator</td><td><code>dynamo-operator.json</code></td><td>DGD 状态、副本数、Reconcile 延迟</td><td>"运维看板" — 看看 K8s 控制面在干什么</td></tr>
<tr><td>非聚合推理</td><td><code>disagg-dashboard.json</code></td><td>Prefill/Decode 分离的延迟和资源使用</td><td>"分拆推理看板" — 看 prefill 和 decode 各自表现</td></tr>
<tr><td>KV Block Manager</td><td><code>kvbm.json</code></td><td>KV block 生命周期、L0/L1/L2 命中率</td><td>"缓存看板" — 看看 KV cache 各层命中率</td></tr>
<tr><td>SGLang</td><td><code>sglang.json</code></td><td>SGLang 后端特有指标</td><td>"SGLang 专用看板"</td></tr>
<tr><td>GPU (DCGM)</td><td><code>dcgm-metrics.json</code></td><td>GPU 利用率、温度、显存、ECC 错误</td><td>"GPU 硬件看板" — 显存、温度、利用率</td></tr>
<tr><td>故障恢复</td><td><code>failover.json</code></td><td>故障次数、恢复时间、级联删除</td><td>"故障看板" — 看系统挂了几次、恢复多快</td></tr>
<tr><td>本地资源</td><td><code>dynamo_local_resource_monitor.json</code></td><td>裸金属部署的资源监控</td><td>"裸金属看板" — CPU/内存/磁盘使用</td></tr>
</table>

<pre><code><span class="cmt"># 如果仪表板未自动加载，手动导入:</span>
<span class="cmt"># 方式 1: 通过 Grafana UI</span>
<span class="cmt"># Grafana → Dashboards → Import → 上传 JSON 文件</span>

<span class="cmt"># 方式 2: 通过 Docker 卷挂载（自动加载）</span>
docker run -d --name grafana \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  -v $(pwd)/dynamo/dev/observability/grafana_dashboards:/var/lib/grafana/dashboards \
  grafana/grafana:latest

<span class="cmt"># 方式 3: 通过 curl API 导入</span>
curl -s -X POST http://admin:admin@localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  --data-binary @dynamo/dev/observability/grafana_dashboards/dynamo.json</code></pre>

<h4>第四步：验证端到端数据流</h4>
<div class="plain">
启动一切之后，你需要验证数据是否正确流动。按照以下 checklist 逐项检查：
</div>

<table>
<tr><th>检查项</th><th>验证命令</th><th>预期结果</th></tr>
<tr><td>Dynamo /metrics 可访问</td><td><code>curl http://localhost:8081/metrics</code></td><td>返回 Prometheus 文本格式指标</td></tr>
<tr><td>Prometheus 抓取成功</td><td>打开 http://localhost:9090 → Status → Targets</td><td>所有 target 状态为 UP</td></tr>
<tr><td>Prometheus 有数据</td><td>http://localhost:9090 → 搜索 <code>dynamo_frontend_requests_total</code></td><td>显示时间序列数据</td></tr>
<tr><td>Grafana 数据源连通</td><td>Grafana → Data Sources → Prometheus → Save &amp; Test</td><td>"Data source is working"</td></tr>
<tr><td>Grafana 仪表板显示</td><td>打开导入的仪表板</td><td>图表有数据、非空白</td></tr>
<tr><td>DCGM GPU 指标</td><td>Prometheus → 搜索 <code>DCGM_FI_DEV_GPU_UTIL</code></td><td>GPU 利用率数据</td></tr>
</table>

<h4>Dynamo 双通道指标架构</h4>
<div class="plain">
Dynamo 的指标系统有两个"面"：一个是 expfmt（文本格式），用于 Prometheus HTTP 拉取；另一个是 typed（结构化数据），用于 OTLP gRPC 推送。两者底层数据源相同，但避免了把 Prometheus 文本再解析回结构化数据的反模式。你可以只接 Prometheus（用 Pull），或者同时接 OTLP（用 Push）到任意支持 OTLP 的后端。
</div>

<p>&lt;<SVG_BLOCK_6>&gt;</p>
<h4>Python 端的指标注册</h4>
<div class="plain">
vLLM/SGLang 引擎启动时，会自动通过 PyO3 桥将 Python 的 prometheus_client 指标注册到 Rust 端的 MetricsRegistry。一个指标同时注册到两个通道：expfmt（Prometheus 文本拉取）和 typed（OTLP 结构化推送）。
</div>

<pre><code><span class="cmt"># dynamo/components/src/dynamo/common/utils/prometheus.py:55</span>
<span class="kw">def</span> <span class="fn">register_engine_metrics_callback</span>(endpoint, registry, auto_labels=None, prefix=None):
    <span class="str">"""注册引擎指标到两个表面。"""</span>
    <span class="cmt"># 1. expfmt: 用于 Prometheus HTTP 拉取</span>
    endpoint.metrics.<span class="fn">register_prometheus_expfmt_callback</span>(
        <span class="kw">lambda</span>: get_prometheus_expfmt(registry, prefix))
    <span class="cmt"># 2. typed: 用于 OTLP gRPC 推送</span>
    endpoint.metrics.<span class="fn">register_prometheus_typed_callback</span>(
        <span class="kw">lambda</span>: get_prometheus_typed(registry, prefix))
</code></pre>

<h4>可用指标速查</h4>
<table>
<tr><th>指标名</th><th>类型</th><th>含义</th><th>关键标签</th><th>大白话</th></tr>
<tr><td><code>dynamo_frontend_requests_total</code></td><td>Counter</td><td>总请求数</td><td>model, worker_id, finish_reason</td><td>"一共处理了多少请求"</td></tr>
<tr><td><code>dynamo_frontend_inflight_requests</code></td><td>Gauge</td><td>当前处理中的请求数</td><td>model, worker_id</td><td>"现在正在处理多少请求"</td></tr>
<tr><td><code>dynamo_frontend_time_to_first_token_seconds</code></td><td>Histogram</td><td>首 token 延迟（TTFT）</td><td>model, worker_id</td><td>"用户等了多久看到第一个字"</td></tr>
<tr><td><code>dynamo_frontend_inter_token_latency_seconds</code></td><td>Histogram</td><td>Token 间延迟（ITL）</td><td>model, worker_id</td><td>"两个字之间隔了多久"</td></tr>
<tr><td><code>dynamo_component_kv_cache_hit_rate</code></td><td>Gauge</td><td>KV Cache 命中率</td><td>dp_rank, worker_id</td><td>"缓存命中了还是得重新算"</td></tr>
<tr><td><code>dynamo_component_gpu_cache_usage_percent</code></td><td>Gauge</td><td>GPU 缓存使用率</td><td>dp_rank, worker_id</td><td>"GPU 显存里的缓存占了多少"</td></tr>
<tr><td><code>dynamo_router_inflight_requests</code></td><td>Gauge</td><td>路由层 inflight 数</td><td>router_id</td><td>"路由器正在处理多少请求"</td></tr>
<tr><td><code>dynamo_planner_num_prefill_replicas</code></td><td>Gauge</td><td>当前 Prefill 副本数</td><td>—</td><td>"Planner 决定跑几个 prefill"</td></tr>
<tr><td><code>dynamo_planner_observed_ttft_ms</code></td><td>Gauge</td><td>观测到的 TTFT</td><td>—</td><td>"Planner 看到的实际延迟"</td></tr>
<tr><td><code>dynamo_tokio_worker_busy_ratio</code></td><td>Gauge</td><td>Tokio worker 忙度</td><td>—</td><td>"Rust 异步运行时忙不忙"</td></tr>
<tr><td><code>dynamo_routing_overhead_scheduling_ms</code></td><td>Histogram</td><td>路由调度耗时</td><td>—</td><td>"路由决策花了多久"</td></tr>
</table>

<h3 id="tracing">2. Tracing 链路追踪 — 从配置到查看的完整步骤 <span class="badge badge-rust">Rust</span> <span class="badge badge-py">Python</span></h3>

<div class="plain">
Tracing 的大白话解释：当一个请求进入系统，前端创建一个"根 span"（就像给这个请求发了一个身份证）。然后每经过一个组件（prefill worker → decode worker），都会创建一个子 span（"我也是这个请求的一部分"）。这些 span 通过 W3C traceparent header 跨进程传播 — 就像接力赛跑，每个跑者手里拿着同一个接力棒。最后所有 span 通过 OTLP 协议推送到 Tempo 存储，你可以在 Grafana 里看到完整的请求链路树，精确到每个环节的耗时。
</div>

<h4>第一步：配置 Tracing 输出</h4>

<h5>5A. 环境变量配置</h5>
<div class="plain">
Dynamo 的 tracing 初始化是幂等的（使用 Once 保证只执行一次）。你通过环境变量控制日志级别和 OTel 导出行为。
</div>

<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>DYN_LOG</code></td><td><code>warn</code></td><td>tracing-subscriber 的 EnvFilter，控制日志级别</td><td>"日志详细程度"</td></tr>
<tr><td><code>OTEL_EXPORT_ENABLED</code></td><td><code>0</code></td><td>设为 <code>1</code> 启用 OTel 导出</td><td>"是否启用链路追踪"</td></tr>
<tr><td><code>OTEL_EXPORTER_OTLP_ENDPOINT</code></td><td><code>http://localhost:4317</code></td><td>OTLP gRPC 端点地址</td><td>"追踪数据发到哪"</td></tr>
<tr><td><code>OTEL_SERVICE_NAME</code></td><td><code>dynamo</code></td><td>服务名称，用于区分不同组件</td><td>"这个组件叫什么名字"</td></tr>
<tr><td><code>OTEL_RESOURCE_ATTRIBUTES</code></td><td>空</td><td>附加属性，如 <code>deployment.environment=staging</code></td><td>"额外标签"</td></tr>
</table>

<pre><code><span class="cmt"># 启动 Dynamo 组件时设置环境变量</span>
<span class="kw">export</span> DYN_LOG=<span class="str">info,dynamo=debug</span>        <span class="cmt"># 总体 info，dynamo 模块 debug</span>
<span class="kw">export</span> OTEL_EXPORT_ENABLED=<span class="str">1</span>           <span class="cmt"># 启用 OTel 导出</span>
<span class="kw">export</span> OTEL_EXPORTER_OTLP_ENDPOINT=<span class="str">http://otel-collector:4317</span>
<span class="kw">export</span> OTEL_SERVICE_NAME=<span class="str">dynamo-frontend</span>  <span class="cmt"># 或 dynamo-backend</span>

<span class="cmt"># 启动组件</span>
python -m dynamo.frontend --router-mode kv</code></pre>

<h5>5B. Tracing 初始化流程</h5>
<div class="call-chain">
<span class="fn">init()</span> <span style="color:var(--dim)">(Once 保证幂等 — 多次调用安全)</span>
  <span class="arrow">→</span> <span class="fn">setup_logging()</span> <span style="color:var(--dim)">(logging.rs)</span>
  <span class="arrow">→</span> 读 <code>DYN_LOG</code> 环境变量构建 EnvFilter
  <span class="arrow">→</span> 创建 Layered Subscriber:
    <span style="color:var(--dim)">├── fmt layer (stdout 输出)</span>
    <span style="color:var(--dim)">├── EnvFilter (按 DYN_LOG 过滤)</span>
  <span class="arrow">→</span> 若 <code>OTEL_EXPORT_ENABLED=1</code>:
    <span style="color:var(--dim)">├── 创建 SdkTracerProvider (opentelemetry-sdk)</span>
    <span style="color:var(--dim)">├── 创建 OTLP exporter (gRPC → OTEL_EXPORTER_OTLP_ENDPOINT)</span>
    <span style="color:var(--dim)">├── 安装 tracing-opentelemetry layer (Rust tracing span → OTel span 桥接)</span>
  <span class="arrow">→</span> 安装 DistributedTraceIdLayer:
    <span style="color:var(--dim)">└── 每个 span 自动注入 trace_id / span_id 到日志行</span>
</div>

<h4>第二步：跨进程 Trace 传播</h4>
<div class="plain">
W3C Trace Context 是业界标准。Dynamo 使用 <code>traceparent</code> 和 <code>tracestate</code> HTTP headers 在组件间传播 trace context。当 frontend 收到一个请求，它检查请求是否已携带 traceparent：如果有，说明上游（如 API Gateway）已经创建了 trace，Dynamo 作为子 span 加入；如果没有，Dynamo 创建一个新的根 trace。
</div>

<div class="call-chain">
<span style="color:var(--dim)">请求到达 Frontend:</span>
  <span class="arrow">→</span> 检查 <code>traceparent</code> header
    <span style="color:var(--dim)">├── 有 → 提取 trace_id + parent_span_id，创建子 span</span>
    <span style="color:var(--dim)">└── 无 → 生成新 trace_id，创建根 span "http-request"</span>
  <span class="arrow">→</span> Frontend 路由请求到 Worker:
    <span style="color:var(--dim)">├── 将当前 span 的 traceparent 写入 NATS message header</span>
    <span style="color:var(--dim)">└── Worker 从 header 提取 context，创建子 span "handle_payload"</span>
  <span class="arrow">→</span> Prefill → Decode (Disagg 模式):
    <span style="color:var(--dim)">└── 同样通过 header 传播，保持同一 trace_id</span>
</div>

<h4>第三步：启动 Tracing 后端</h4>

<h5>3A. 使用 Docker 一键启动（包含 Tempo）</h5>
<div class="plain">
docker-observability.yml 已经包含了 Tempo（追踪存储）和 OTel Collector（数据中转）。一键启动即可。
</div>

<pre><code><span class="cmt"># 启动完整可观测性栈（含 Tempo + OTel Collector）</span>
docker compose -f dynamo/dev/observability/docker-observability.yml up -d

<span class="cmt"># 验证 Tempo 运行</span>
curl http://localhost:3200/health

<span class="cmt"># 验证 OTel Collector 运行</span>
curl http://localhost:13133</code></pre>

<h5>3B. OTel Collector 配置</h5>
<div class="plain">
OTel Collector 是"数据中转站"。Dynamo 组件把 tracing data 发给 Collector，Collector 再转发给 Tempo 存储。你不需要每个 Dynamo 组件直连 Tempo。
</div>

<pre><code><span class="cmt"># otel-collector-config.yaml (docker-observability.yml 中内嵌)</span>
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]</code></pre>

<h5>3C. Grafana 中查看 Traces</h5>
<table>
<tr><th>步骤</th><th>操作</th><th>说明</th></tr>
<tr><td>①</td><td>打开 Grafana → Explore</td><td>—</td></tr>
<tr><td>②</td><td>数据源选择 Tempo</td><td>—</td></tr>
<tr><td>③</td><td>点击 "Search" 标签</td><td>按时间/服务名筛选 traces</td></tr>
<tr><td>④</td><td>点击任一 trace 条目</td><td>进入 "Trace Graph" 查看完整 span 树</td></tr>
<tr><td>⑤</td><td>展开 span 查看 details</td><td>每个 span 的耗时、标签、日志</td></tr>
</table>

<h4>初始化链</h4>
<div class="call-chain">
<span class="fn">init()</span> <span style="color:var(--dim)">(Once 保证幂等)</span>
  <span class="arrow">→</span> <span class="fn">setup_logging()</span>
  <span class="arrow">→</span> 读 <span class="str">DYN_LOG</span> 环境变量构建 EnvFilter
  <span class="arrow">→</span> 若 <span class="str">OTEL_EXPORT_ENABLED=1</span> → 创建 SdkTracerProvider + OTLP exporter
  <span class="arrow">→</span> 安装 tracing-opentelemetry layer
  <span class="arrow">→</span> 安装 DistributedTraceIdLayer — 每 span 自动注入 trace_id/span_id
</div>

<h4 id="tracing-diag">图解：跨进程 Trace 传播</h4>

<p>&lt;<SVG_BLOCK_7>&gt;</p>
<h3 id="health">3. Health 健康检查 <span class="badge badge-rust">Rust</span></h3>

<div class="plain">
Dynamo 有两个健康检查端点，大白话解释：<code>/live</code> 是"你还活着吗？"——进程还在跑、没在关机就回答"活着"；<code>/health</code> 是"你能干活吗？"——不仅要活着，还要能处理请求。Canary（金丝雀）机制：每个 endpoint 有一个后台"巡逻员"，如果一段时间没收到真实请求，它就主动发个"试探包"看看后端能不能响应。如果试探包超时，说明后端有问题，/health 返回 503。
</div>

<h4>端点说明</h4>
<table>
<tr><th>端点</th><th>类型</th><th>检查内容</th><th>返回</th><th>大白话</th></tr>
<tr><td><code>/live</code></td><td>Liveness</td><td>进程是否在 cancel/shutdown</td><td>200 live / 503 shutting_down</td><td>"进程还在吗？"</td></tr>
<tr><td><code>/health</code></td><td>Readiness</td><td>Canary 通过 + 所有 endpoint 就绪 + 无 ReadinessHold</td><td>200 healthy / 503 not_ready</td><td>"能干活吗？"</td></tr>
</table>

<h4>如何使用健康检查</h4>

<h5>K8s 环境</h5>
<div class="plain">
在 K8s 的 Deployment 中配置 livenessProbe 和 readinessProbe。liveness 决定重启，readiness 决定流量。Dynamo 的 Helm chart 已经预配置了这些探针。
</div>

<pre><code><span class="cmt"># K8s Deployment 中的健康检查配置（Helm chart 已预配置）</span>
livenessProbe:
  httpGet:
    path: /live
    port: 8081    <span class="cmt"># DYN_SYSTEM_PORT</span>
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: 8081
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1</code></pre>

<h5>裸金属环境</h5>
<div class="plain">
非 K8s 部署可以用简单的 shell 脚本定期检查健康状态，发现问题就重启进程。
</div>

<pre><code><span class="cmt"># 裸金属健康检查脚本</span>
<span class="kw">#!/bin/bash</span>
SYSTEM_PORT=<span class="num">8081</span>
MAX_RETRIES=<span class="num">3</span>

<span class="kw">for</span> i in <span class="str">$(seq 1 $MAX_RETRIES)</span>; <span class="kw">do</span>
  status=$(curl -s -o /dev/null -w <span class="str">"%{http_code}"</span> http://localhost:$SYSTEM_PORT/health)
  <span class="kw">if</span> [ "$status" = <span class="str">"200"</span> ]; <span class="kw">then</span>
    echo <span class="str">"OK - healthy"</span>
    exit 0
  fi
  echo <span class="str">"WARN - health check failed (HTTP $status), attempt $i/$MAX_RETRIES"</span>
  sleep 5
done

<span class="cmt"># 连续失败 → 重启进程</span>
echo <span class="str">"CRITICAL - restarting worker"</span>
systemctl restart dynamo-worker</code></pre>

<h4>Canary 探测流程</h4>
<div class="call-chain">
每 endpoint 一个 tokio task:
  <span class="fn">select!</span> {
    <span class="fn">sleep(CANARY_WAIT_TIME)</span>  <span class="arrow">→</span> 空闲超时 → 发送 canary payload → 成功:Ready / 超时:NotReady
    <span class="fn">notifier.notified()</span>   <span class="arrow">→</span> 检测到请求活动 → 立即标记 Ready，重置计时器
  }
</div>

<h4>ReadinessHold RAII 守卫</h4>
<div class="plain">
ReadinessHold 是"启动闸门"。模型加载、引擎初始化等耗时操作期间，endpoint 强制处于 NotReady 状态，不会接收任何流量。初始化完成后 drop 这个 guard，endpoint 才能转为 Ready。
</div>

<pre><code><span class="cmt">// system_health.rs — 启动阶段持有此 guard → endpoint 强制 NotReady</span>
<span class="kw">let</span> hold = SystemHealth.<span class="fn">hold_endpoint_readiness</span>(<span class="str">"endpoint-name"</span>);
<span class="cmt">// ... 模型加载、初始化 ...</span>
<span class="kw">drop</span>(hold);  <span class="cmt">// → 释放 hold，endpoint 可转为 Ready，流量可以进来了</span></code></pre>

<h4>健康检查与扩缩容的关联</h4>
<div class="plain">
健康检查不仅仅是监控，它还直接影响扩缩容行为。当 K8s 检测到某个 Pod 的 /health 返回 503，会将其从 Service 的 Endpoints 列表中移除，流量不再发往该 Pod。如果 /live 返回 503，K8s 会直接重启该 Pod。Planner 的扩缩容决策也依赖于健康状态：不健康的 worker 不会被计入"实际可用数量"。
</div>

<h3 id="fpm">4. FPM 性能指标 — 扩缩容的数据源 <span class="badge badge-py">Python</span></h3>

<div class="plain">
FPM（Forward Pass Metrics）是扩缩容决策的"眼睛"。大白话：每次 GPU 完成一次推理（forward pass），vLLM 就会"量"一下这次推理的数据 — 处理了多少 token、花了多少时间、队列里还剩多少请求等着。这些量化数据通过 ZMQ 发送给 Planner。Planner 用这些数据训练回归模型，预测"如果加一台机器，TTFT 会改善多少"。没有 FPM，Planner 就是瞎子 — 它不知道当前集群的性能状况，只能瞎猜。
</div>

<div class="plain" style="border-color: rgba(248,81,73,.4); background: linear-gradient(135deg, rgba(248,81,73,.08), rgba(210,153,34,.08));">
⚠️ <strong>FPM 默认也是关闭的！</strong><br>
需要通过 <code>--fpm-trace</code> CLI 参数或 <code>DYN_FPM_TRACE=1</code> 环境变量显式启用。不启用则不会注入 <code>InstrumentedScheduler</code>，Planner 收不到 FPM 数据，负载扩缩（load scaling）不会生效。
</div>

<h4>FPM 采集与配置</h4>

<h5>启用方式</h5>
<pre><code><span class="cmt"># 方式 1: CLI 参数</span>
DYN_SYSTEM_PORT=<span class="num">8081</span> python -m dynamo.vllm --model $MODEL --fpm-trace

<span class="cmt"># 方式 2: 环境变量</span>
DYN_SYSTEM_PORT=<span class="num">8081</span> DYN_FPM_TRACE=<span class="num">1</span> python -m dynamo.vllm --model $MODEL</code></pre>

<h5>采集器：InstrumentedScheduler</h5>
<div class="plain">
FPM 数据采集器是 vLLM 调度器的一个"插桩"版本。它在标准调度器的每次调度循环后，提取本次 forward pass 的统计数据。实现上使用了 WelfordAccumulator 来计算 token 长度的方差（在线算法，不需要存储所有历史值）。
</div>

<div class="call-chain">
<span class="hl-yellow">InstrumentedScheduler</span> <span style="color:var(--dim)">(instrumented_scheduler.py)</span>
  <span class="arrow">→</span> 继承 vLLM 标准 Scheduler
  <span class="arrow">→</span> 每次调度循环后调用 <span class="fn">_update_from_output()</span>
  <span class="arrow">→</span> 提取: 处理的请求数、token 数、forward pass 耗时、队列深度
  <span class="arrow">→</span> WelfordAccumulator 计算均值和方差
  <span class="arrow">→</span> 组装 <span class="fn">ForwardPassMetrics</span> 结构体
  <span class="arrow">→</span> msgpack 序列化 → FpmPublisherThread 后台发送
</div>

<h5>ZMQ 传输</h5>
<div class="plain">
FPM 数据通过 ZeroMQ 的 PUB/SUB 模式传输。每个 dp_rank 一个 PUB socket（端口从 20380 开始递增），Planner 作为 SUB 订阅者连接所有 PUB socket。如果 worker 空闲（没有 forward pass），FpmPublisherThread 每秒发一次心跳，让 Planner 知道 worker 还活着。
</div>

<pre><code><span class="cmt"># FPM 传输端口配置</span>
<span class="cmt"># 每个 worker 的每个 dp_rank 一个端口</span>
<span class="cmt"># 格式: tcp://*:{DYN_FORWARDPASS_METRIC_PORT + dp_rank}</span>
<span class="cmt"># DYN_FORWARDPASS_METRIC_PORT 默认 20380 (envs.py:20)</span>
<span class="cmt"># 例: dp_rank=0 → 20380, dp_rank=1 → 20381</span>

<span class="cmt"># Planner 端订阅配置</span>
<span class="cmt"># FpmEventSubscriber 自动连接所有 worker 的 FPM 端口</span>
<span class="cmt"># 按 (worker_id, dp_rank) 分组，喂给回归模型</span></code></pre>

<h4>FPM 数据结构</h4>
<pre><code><span class="cmt"># msgspec.Struct + msgpack 序列化</span>
<span class="kw">class</span> <span class="fn">ForwardPassMetrics</span>:
    version, worker_id, dp_rank, counter_id
    wall_time: <span class="kw">float</span>              <span class="cmt"># 这次 forward pass 花了多久（秒）</span>
    scheduled_requests: <span class="fn">ScheduledRequestMetrics</span>  <span class="cmt"># 本次调度了多少请求</span>
    queued_requests:    <span class="fn">QueuedRequestMetrics</span>     <span class="cmt"># 队列里还剩多少</span>

<span class="kw">class</span> <span class="fn">ScheduledRequestMetrics</span>:
    num_prefill_requests        <span class="cmt"># Prefill 请求数</span>
    sum_prefill_tokens          <span class="cmt"># Prefill 总 token 数</span>
    var_prefill_length          <span class="cmt"># Prefill 长度方差</span>
    sum_prefill_kv_tokens       <span class="cmt"># Prefill KV token 数</span>
    num_decode_requests         <span class="cmt"># Decode 请求数</span>
    sum_decode_kv_tokens        <span class="cmt"># Decode KV token 数</span>
    var_decode_kv_tokens        <span class="cmt"># Decode KV token 方差</span></code></pre>

<h4>FPM 数据结构</h4>
<pre><code><span class="cmt"># msgspec.Struct + msgpack 序列化</span>
<span class="kw">class</span> <span class="fn">ForwardPassMetrics</span>:
    version, worker_id, dp_rank, counter_id
    wall_time: <span class="kw">float</span>              <span class="cmt"># 这次 forward pass 花了多久</span>
    scheduled_requests: <span class="fn">ScheduledRequestMetrics</span>  <span class="cmt"># 本次调度了多少请求</span>
    queued_requests:    <span class="fn">QueuedRequestMetrics</span>     <span class="cmt"># 队列里还剩多少</span>

<span class="kw">class</span> <span class="fn">ScheduledRequestMetrics</span>:
    num_prefill_requests, sum_prefill_tokens, var_prefill_length
    sum_prefill_kv_tokens, num_decode_requests
    sum_decode_kv_tokens, var_decode_kv_tokens
</code></pre>

<h4 id="fpm-diag">图解：FPM 数据流 — 从 GPU 到扩缩容决策</h4>

<p>&lt;<SVG_BLOCK_8>&gt;</p>
<h3 id="req-trace">5. Request Trace 请求追踪 — 审计日志 <span class="badge badge-rust">Rust</span></h3>

<div class="plain">
Request Trace 和分布式 Tracing 不同。Tracing 回答"这个请求经过了哪些组件、每个组件花了多久"，Request Trace 回答"这个请求的输入输出是什么、性能如何"。大白话：Tracing 是"请求路径图"，Request Trace 是"请求成绩单"。它记录每个请求的完整元数据快照 — 输入输出 token 数、TTFT、ITL、KV 命中率、队列深度等。数据通过广播 bus 分发到多个 sink（文件、NATS、OTLP、S3），每个 sink 一个独立 tokio 任务，互不阻塞。
</div>

<h4>配置与使用</h4>

<h5>5A. 环境变量</h5>
<table>
<tr><th>环境变量</th><th>默认</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>DYN_REQUEST_TRACE</code></td><td><code>0</code></td><td>设为 <code>1</code> 开启追踪</td><td>"要不要记录请求成绩单"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_SINKS</code></td><td><code>file</code></td><td><code>file,stderr,nats,otel,s3</code>，逗号分隔</td><td>"记录存到哪"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_RECORDS</code></td><td><code>request_end,tool</code></td><td>记录类型</td><td>"记录哪些事件"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_FILE_FORMAT</code></td><td><code>jsonl_gz</code></td><td>可选 <code>jsonl</code> / <code>jsonl_gz</code></td><td>"文件格式：纯文本还是压缩"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_FILE_PATH</code></td><td><code>/tmp/request_trace.jsonl.gz</code></td><td>文件输出路径</td><td>"文件存哪"</td></tr>
</table>

<h5>5B. 开启方法</h5>
<pre><code><span class="cmt"># 最小配置：开启追踪，输出到本地文件</span>
<span class="kw">export</span> DYN_REQUEST_TRACE=<span class="num">1</span>
<span class="kw">export</span> DYN_REQUEST_TRACE_SINKS=<span class="str">file</span>
<span class="kw">export</span> DYN_REQUEST_TRACE_FILE_PATH=<span class="str">/var/log/dynamo/request_trace.jsonl.gz</span>

<span class="cmt"># 完整配置：同时输出到文件 + NATS + OTLP</span>
<span class="kw">export</span> DYN_REQUEST_TRACE=<span class="num">1</span>
<span class="kw">export</span> DYN_REQUEST_TRACE_SINKS=<span class="str">file,nats,otel</span>
<span class="kw">export</span> DYN_REQUEST_TRACE_RECORDS=<span class="str">request_end,tool,request_payload</span>
<span class="kw">export</span> DYN_REQUEST_TRACE_FILE_FORMAT=<span class="str">jsonl_gz</span>

<span class="cmt"># 启动 Dynamo</span>
python -m dynamo.frontend --router-mode kv</code></pre>

<h4>数据模型</h4>
<pre><code><span class="kw">struct</span> <span class="fn">RequestTraceRecord</span> {
    event_type: <span class="fn">RequestTraceEventType</span>,  <span class="cmt">// RequestEnd | ToolStart | ToolEnd | RequestPayload</span>
    request:   <span class="fn">Option</span>&lt;RequestTraceMetrics&gt;,
    tool:      <span class="fn">Option</span>&lt;RequestTraceToolEvent&gt;,
    payload:   <span class="fn">Option</span>&lt;RequestTracePayload&gt;,
}
<span class="cmt">// 核心字段: request_id, model, input/output_tokens, ttft_ms, itl_ms,</span>
<span class="cmt">//           kv_hit_rate, queue_depth, worker, finish_reason</span></code></pre>

<h4>架构 — 广播 Bus + 独立 Sink 任务</h4>
<div class="call-chain">
<span class="hl-yellow">TelemetryBus&lt;RequestTraceRecord&gt;</span>  <span style="color:var(--dim)">// broadcast, capacity=1024</span>
  <span class="arrow">├──→</span> <span class="fn">tokio task</span> → JsonlGzipSink  <span style="color:var(--dim)">// 1MB buffer, 1s flush, 256MB roll</span>
  <span class="arrow">├──→</span> <span class="fn">tokio task</span> → NatsSink      <span style="color:var(--dim)">// 发送到 NATS 主题</span>
  <span class="arrow">├──→</span> <span class="fn">tokio task</span> → OtelSink      <span style="color:var(--dim)">// OTLP 导出到追踪后端</span>
  <span class="arrow">└──→</span> <span class="fn">tokio task</span> → S3Sink        <span style="color:var(--dim)">// 上传到对象存储</span>
</div>

<h4>Request Trace vs Tracing vs Metrics</h4>
<table>
<tr><th>维度</th><th>Metrics</th><tr><th>Tracing</th><th>Request Trace</th></tr>
<tr><td>大白话</td><td>"仪表盘" — 聚合统计</td><td>"路径图" — 请求链路</td><td>"成绩单" — 逐请求审计</td></tr>
<tr><td>粒度</td><td>聚合（直方图/计数器）</td><td>单请求（span 树）</td><td>单请求（完整元数据）</td></tr>
<tr><td>存储</td><td>Prometheus（时序 DB）</td><td>Tempo（追踪 DB）</td><td>文件/NATS/S3（原始数据）</td></tr>
<tr><td>用途</td><td>实时监控、告警、扩缩容</td><td>调试延迟、定位瓶颈</td><td>审计、数据分析、计费</td></tr>
<tr><td>开销</td><td>极低（计数/打桶）</td><td>中等（span 创建/导出）</td><td>较高（逐请求记录）</td></tr>
</table>

<h2 class="section-title" id="config-ref">可观测性配置速查</h2>

<div class="plain">
下面是一张"抄作业"表格 — 列出可观测性相关的所有关键环境变量及其默认值。大部分可观测功能默认关闭，需要手动开启。
</div>

<h4>System Server（/metrics、/health、/live）</h4>
<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>DYN_SYSTEM_PORT</code></td><td><code>-1</code>（关闭）</td><td>设为 >=0 开启，0 为随机端口</td><td>"指标服务器端口，默认关着"</td></tr>
<tr><td><code>DYN_SYSTEM_HOST</code></td><td><code>0.0.0.0</code></td><td>绑定地址</td><td>"监听哪个 IP"</td></tr>
<tr><td><code>DYN_SYSTEM_HEALTH_PATH</code></td><td><code>/health</code></td><td>健康检查路径</td><td>—</td></tr>
<tr><td><code>DYN_SYSTEM_LIVE_PATH</code></td><td><code>/live</code></td><td>存活检查路径</td><td>—</td></tr>
<tr><td><code>DYN_SYSTEM_STARTING_HEALTH_STATUS</code></td><td><code>NotReady</code></td><td>初始状态：Ready / NotReady</td><td>"启动时说不说能干活"</td></tr>
</table>

<h4>Health Check（金丝雀探测）</h4>
<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>DYN_HEALTH_CHECK_ENABLED</code></td><td><code>false</code></td><td>启用主动健康检查</td><td>"要不要主动试探后端，默认关着"</td></tr>
<tr><td><code>DYN_CANARY_WAIT_TIME</code></td><td>10s</td><td>空闲等待时间</td><td>"多久没请求就试探一次"</td></tr>
<tr><td><code>DYN_HEALTH_CHECK_REQUEST_TIMEOUT</code></td><td>3s</td><td>单次探测超时</td><td>"试探包等多久算超时"</td></tr>
</table>

<h4>日志（Log）</h4>
<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>DYN_LOG</code></td><td><code>info</code></td><td>日志级别，支持 <code>target=level</code> 语法</td><td>"日志详细程度"</td></tr>
<tr><td><code>DYN_LOGGING_CONSOLE_FORMAT</code></td><td><code>readable</code></td><td>设为 <code>jsonl</code> 输出结构化 JSON 日志</td><td>"日志格式：人看还是机器看"</td></tr>
<tr><td><code>DYN_LOG_USE_LOCAL_TZ</code></td><td>false（UTC）</td><td>使用本地时区</td><td>"日志时间戳用哪个时区"</td></tr>
</table>

<h4>OTLP 导出（Traces + Logs + Metrics）</h4>
<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>OTEL_EXPORT_ENABLED</code></td><td>关闭</td><td>设为 <code>1</code> 启用 traces+logs 导出</td><td>"追踪和日志导出，默认关着"</td></tr>
<tr><td><code>OTEL_METRICS_EXPORTER</code></td><td>关闭</td><td>设为 <code>otlp</code> 启用 metrics 导出</td><td>"指标 OTLP 推送，默认关着"</td></tr>
<tr><td><code>OTEL_EXPORTER_OTLP_ENDPOINT</code></td><td><code>http://localhost:4317</code></td><td>OTLP gRPC 端点</td><td>"数据发到哪"</td></tr>
<tr><td><code>OTEL_SERVICE_NAME</code></td><td><code>dynamo</code></td><td>服务名称</td><td>"这个组件叫什么"</td></tr>
<tr><td><code>OTEL_METRIC_EXPORT_INTERVAL</code></td><td>60000ms（60s）</td><td>指标推送间隔（毫秒）</td><td>"多久推一次指标"</td></tr>
<tr><td><code>OTEL_TRACES_SAMPLE_RATIO</code></td><td>1.0（全采）</td><td>采样率 0.0-1.0</td><td>"抓多少追踪数据"</td></tr>
</table>

<h4>Request Trace（请求审计日志）</h4>
<table>
<tr><th>环境变量</th><th>默认值</th><th>说明</th><th>大白话</th></tr>
<tr><td><code>DYN_REQUEST_TRACE</code></td><td>关闭</td><td>设为 <code>1</code> 开启</td><td>"逐请求审计，默认关着"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_SINKS</code></td><td><code>file</code></td><td><code>file,stderr,nats,otel,s3</code></td><td>"记录存到哪"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_RECORDS</code></td><td><code>request_end,tool</code></td><td>记录类型</td><td>"记哪些事件"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_FILE_PATH</code></td><td><code>/tmp/dynamo-request-trace</code></td><td>文件输出路径</td><td>"文件存哪"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_FILE_FORMAT</code></td><td><code>jsonl_gz</code></td><td><code>jsonl</code> / <code>jsonl_gz</code></td><td>"纯文本还是压缩"</td></tr>
<tr><td><code>DYN_REQUEST_TRACE_CAPACITY</code></td><td>1024</td><td>内存队列容量，满了会丢数据</td><td>"排队 buffer 多大"</td></tr>
</table>

<h4>Frontend 指标直方图桶</h4>
<div class="plain">
Frontend 的直方图指标（TTFT、ITL、请求时长等）支持自定义桶边界。通过 <code>DYN_METRICS_TTFT_MIN</code>、<code>DYN_METRICS_TTFT_MAX</code>、<code>DYN_METRICS_TTFT_COUNT</code> 等环境变量控制。类似地还有 <code>DYN_METRICS_ITL_*</code>、<code>DYN_METRICS_REQUEST_DURATION_*</code>、<code>DYN_METRICS_INPUT_SEQUENCE_*</code>、<code>DYN_METRICS_OUTPUT_SEQUENCE_*</code>。
</div>

<h4>一键开启所有可观测性（推荐开发环境）</h4>
<pre><code><span class="cmt"># 开启 /metrics + /health + /live</span>
<span class="kw">export</span> DYN_SYSTEM_PORT=<span class="num">8081</span>

<span class="cmt"># 开启 OTel Traces + Logs 导出</span>
<span class="kw">export</span> OTEL_EXPORT_ENABLED=<span class="num">1</span>
<span class="kw">export</span> OTEL_EXPORTER_OTLP_ENDPOINT=<span class="str">http://otel-collector:4317</span>
<span class="kw">export</span> OTEL_SERVICE_NAME=<span class="str">dynamo-backend</span>

<span class="cmt"># 开启 OTel Metrics 推送（可选，Prometheus Pull 已够用的话不需要）</span>
<span class="kw">export</span> OTEL_METRICS_EXPORTER=<span class="str">otlp</span>

<span class="cmt"># 开启结构化 JSON 日志（可选，方便 Loki 采集）</span>
<span class="kw">export</span> DYN_LOGGING_CONSOLE_FORMAT=<span class="str">jsonl</span>

<span class="cmt"># 开启金丝雀健康检查（可选）</span>
<span class="kw">export</span> DYN_HEALTH_CHECK_ENABLED=<span class="num">1</span>

<span class="cmt"># 启动 Dynamo worker</span>
DYN_SYSTEM_PORT=<span class="num">8081</span> python -m dynamo.vllm --model $MODEL</code></pre>

<div class="summary-box">
  <p><strong>总结:</strong></p>
  <p><strong>扩缩容:</strong> 四层联动 — K8s Adapter 管 Pod（唯一入口防冲突），Planner 管决策（SLA 回归预测 + 缩容安全校验），KV Hash Ring 管数据迁移（两阶段 drain + 热数据预热），FPM 管数据源（GPU → ZMQ → 回归模型）。扩缩策略分 agg（组合）/ disagg（独立 prefill/decode）/ single（单一组件），算法分 Easy（静态阈值）和 SLA（回归模型预测）两档，粒度为每 5s ±1 副本。</p>
  <p><strong>通信域:</strong> TP/PP 固定（固定生命周期），EP 弹性（支持 live resize）。故障时 inter-pod 级联删除全组（NCCL 无法部分恢复）、intra-pod 单 rank 恢复（standby 接管 + GMS 加速）。</p>
  <p><strong>非 K8s 扩缩容:</strong> 通过 PlannerConnector 抽象层切换到 VirtualConnector，etcd 协调（Coordinator 写决策 → Client 消费决策 → 执行启停），完整协议包含 decision_id/scaled_decision_id 握手机制。</p>
  <p><strong>可观测性:</strong> 三管齐下 — Metrics（Prometheus Pull / OTLP Push 双通道，四层层次结构自动标签，10+ Grafana 仪表板）、Tracing（W3C traceparent 跨进程传播，Tempo 存储，Grafana 查看）、Request Trace（逐请求审计日志，广播 Bus 多 Sink）。FPM 是扩缩容的数据源，从 GPU 到决策的完整链路：InstrumentedScheduler → ZMQ → Planner → 回归模型 → 扩缩容决策。</p>
</div>