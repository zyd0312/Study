
面向存证的区块链，图灵完备的执行引擎。1. 对虚拟机的调用 2.获取对上下文的获取 3. 内置合约，为了支持跨合约调用，会适配外部模块，拖慢速度。账户体系和智能合约的执行相辅相成，面对大量存证的场景（联盟链），优化存证路径。
参考公链：1. 时刻运行的检测者 2. 零知识证明
想法:1. 存证场景的功能需求（hyperchain，长安链） 2. 现有公链的layer1layer2 rollup 的思路，看看是否能从公链借鉴来解决联盟链存证的场景 3.针对这个场景设计联盟链的结构

查看公链的应用方案博客，具体采用了哪些标准措施。
当前存证场景到底有哪些功能，结合实现方案

## 当前存证场景功能与实现方案调研

### 1. 存证场景的核心目标

存证系统不是简单把文件放到链上，而是要证明一条电子证据在法律或审计语境下具备以下事实：

- 谁提交或采集了证据；
- 证据内容是什么，之后是否被篡改；
- 证据在什么时间形成、固定、流转；
- 证据来自哪个业务系统、设备、网页或第三方平台；
- 证据原文在哪里保存，如何验证原文与链上记录一致；
- 证据在取证、保管、出证、质证过程中的操作链路是否完整。

最高人民法院《最高人民法院关于互联网法院审理案件若干问题的规定》第十一条明确提到，电子数据真实性审查会关注生成、收集、存储、传输环境，生成主体和时间，保管方式，提取和固定方式，内容是否完整，以及是否可验证；电子签名、可信时间戳、哈希值校验、区块链等技术手段能够证明真实性的，法院应当确认。

因此，面向存证的联盟链应围绕“可信采集 + 哈希锚定 + 时间证明 + 身份签名 + 链下原文保管 + 可验证出证 + 全流程审计”设计，而不是优先追求通用图灵完备合约。

### 2. 当前存证场景常见功能

| 功能 | 业务含义 | 实现方案 |
| --- | --- | --- |
| 身份认证 | 确认提交人、采集人、机构、节点身份 | 个人/企业实名、CA 证书、公钥地址、DID；联盟链节点采用组织证书和角色权限；链上交易必须带签名 |
| 权限控制 | 控制谁能上链、查询、出证、管理节点 | RBAC/ACL；按个人、企业、法院、公证处、仲裁、平台方、审计方划分角色 |
| 证据采集 | 从网页、App、业务系统、IoT、合同、图片、视频等来源采集数据 | SDK/API 接入业务系统；可信浏览器取证；本地文件上传；设备日志采集；采集端生成哈希和元数据 |
| 哈希存证 | 证明某份数据在某时刻已经存在且未被修改 | 原文不上链，只上链 `hash + metadataHash + owner + timestamp + source`；原文存对象存储/归档系统/IPFS/私有文件库 |
| 时间证明 | 证明固定时间 | 链上区块时间 + 节点共识时间；必要时接入可信时间戳服务；出证时展示区块高度、交易哈希、时间戳 |
| 批量存证 | 面向高频业务降低链上压力 | 多条证据先构造 Merkle Tree，只把 Merkle Root 上链；每条证据保留 Merkle Proof |
| 原文保管 | 保存文件、合同、视频、网页快照等大对象 | 链下加密存储，链上保存内容哈希、存储 URI、加密策略、访问策略；大文件分片后生成根哈希 |
| 存证凭证 | 给用户返回可验证凭证 | 凭证包含 evidenceId、txId、blockHeight、dataHash、metadataHash、时间、主体签名、链标识 |
| 验证核验 | 证明提交的原文与链上记录一致 | 重新计算原文哈希，与链上 hash 比对；批量模式下同时校验 Merkle Proof |
| 证据链/流转 | 记录证据被查看、下载、转交、出证、作废等过程 | 设计 `EvidenceRecord` 和 `EvidenceOperation` 两类记录；每次操作写入审计日志，形成 chain of custody |
| 出证服务 | 面向法院、公证处、仲裁、审计输出报告 | 生成 PDF/JSON 证据报告；报告中包含链上交易、哈希算法、签名、原文摘要、核验步骤 |
| 隐私保护 | 避免证据原文和敏感元数据泄露 | 原文链下加密；链上只放摘要；敏感字段脱敏或加密；权限查询；必要时使用零知识证明证明某条件成立 |
| 司法/公证对接 | 提高证据采信和使用效率 | 对接互联网法院诉讼平台、公证处、仲裁委、司法鉴定机构；节点联盟中引入权威机构见证 |
| 监控与审计 | 证明系统持续可靠运行 | 节点状态监控、交易吞吐、失败率、区块延迟、操作日志、安全审计、异常告警 |

### 3. 推荐的数据模型

存证场景不需要每次都调用复杂智能合约。可以把存证设计成“内置系统合约/原生交易类型”，减少 VM 调用和跨合约开销。

```text
EvidenceRecord
  evidenceId          证据唯一编号
  owner               证据所有者/提交主体
  sourceType          file / url / api / iot / contract / video / image
  sourceId            业务系统编号、设备编号、URL、合同号等
  hashAlgo            SHA-256 / SM3
  dataHash            原文哈希
  metadataHash        元数据哈希
  storageUriHash      链下存储地址的哈希，避免泄露真实地址
  timestamp           链上确认时间
  submitterSignature  提交方签名
  sourceSignature     来源系统签名，可选
  prevEvidenceId      上一版本证据，可选
  status              active / revoked / superseded
```

```text
EvidenceOperation
  opId
  evidenceId
  operator
  opType              create / verify / view / export / transfer / revoke
  opTime
  opHash              操作上下文哈希
  signature
```

批量存证时可使用：

```text
BatchEvidenceRoot
  batchId
  merkleRoot
  evidenceCount
  hashAlgo
  batchTimestamp
  submitter
  signature
```

单条证据不上链完整元数据，而是保存链下明细和 Merkle Proof。链上只保存批次根，可以显著降低共识和存储压力。

### 4. 面向联盟链的实现架构

```text
业务系统 / 取证工具 / 用户端
        |
        v
存证接入网关
  - 身份认证
  - 文件哈希
  - 元数据规范化
  - 敏感字段脱敏
  - 批量聚合
        |
        +--------------------+
        |                    |
        v                    v
链下证据库 / 对象存储       联盟链存证服务
  - 原文加密保存             - 原生存证交易
  - 文件分片                 - Merkle Root 上链
  - 访问控制                 - 交易回执
  - 归档备份                 - 区块确认
        |                    |
        +---------+----------+
                  v
核验 / 出证 / 审计服务
  - 哈希核验
  - Merkle Proof 核验
  - 操作链路审计
  - PDF/JSON 出证报告
  - 法院/公证/仲裁接口
```

### 5. 对长安链、趣链和公链 Rollup 思路的借鉴

长安链提供了适合联盟链存证的基础能力：数字证书/公钥身份模型、RBAC 权限控制、Raft/TBFT/MaxBFT 等共识、区块头中的交易 Merkle Root、区块时间戳、区块哈希、交易读写集根，以及区块、交易、状态、历史、事件等多类账本数据存储。它还提供链下扩容方案，思路是把多次交易放到链下处理，最后提交阶段性状态或检查点到链上。

趣链 FiloInk 的产品定位更接近具体存证业务，覆盖司法取证、知识产权保护、公证、数据托管等场景，并强调证据采集、保管、验证、出证以及司法节点联盟见证。Hyperchain 底层则强调联盟链性能、权限、安全、隐私、混合存储、大文件可信存储、多级权限管理和审计能力。

公链 Rollup 的可借鉴点不是把联盟链直接改成 L2，而是借鉴“链下执行/聚合，链上只保存可验证承诺”的思想：

- 高频存证请求先进入链下批处理队列；
- 每批证据生成 Merkle Root；
- 链上只提交批次根、批次元数据和提交方签名；
- 用户持有单条证据的 Merkle Proof；
- 核验时用 `原文 hash + proof + 链上 root` 证明该证据属于某个已确认批次；
- 对关键批次可周期性锚定到更高可信层，例如法院链、行业链、甚至公链，只保存根哈希，不泄露原文。

### 6. 适合存证场景的链结构建议

1. 共识层优先使用联盟链 BFT 或 CFT 共识。若节点都是强监管机构和可信企业，Raft 类共识性能更高；若需要容忍恶意节点，使用 TBFT/MaxBFT/RBFT 类共识。

2. 执行层弱化通用 VM。存证主路径采用内置系统合约或原生交易，支持 `createEvidence`、`batchAnchor`、`verifyEvidence`、`appendOperation`、`revokeEvidence`。复杂业务逻辑再走普通智能合约。

3. 存储层采用“链上摘要 + 链下原文”。链上保存哈希、签名、时间、状态、Merkle Root；链下保存加密原文、网页快照、视频、图片、日志、取证过程包。

4. 数据结构支持批量证明。区块内部已有交易 Merkle Root，业务层再增加 evidence Merkle Root，可以把大量小存证聚合成少量链上交易。

5. 节点角色分层。共识节点由法院、公证处、仲裁委、监管机构、核心平台方运行；同步/见证节点由审计机构、行业协会、大型企业运行；轻节点用于用户核验。

6. 出证服务标准化。出证报告必须包含证据摘要、哈希算法、链 ID、交易 ID、区块高度、区块时间、提交主体、节点签名、核验步骤、原文保管位置说明。

7. 隐私和合规优先。链上不存个人隐私、商业秘密和原文；敏感元数据做哈希或加密；访问原文需要授权；删除需求通过链下原文删除/密钥销毁实现，链上只保留不可逆摘要。

### 7. 参考文档网址

- 最高人民法院：《最高人民法院关于互联网法院审理案件若干问题的规定》  
  https://www.court.gov.cn/fabu/xiangqing/116981.html
- 长安链文档：整体架构说明  
  https://docs.chainmaker.org.cn/tech/%E6%95%B4%E4%BD%93%E8%AF%B4%E6%98%8E.html
- 长安链文档：身份权限管理  
  https://docs.chainmaker.org.cn/tech/%E8%BA%AB%E4%BB%BD%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86.html
- 长安链文档：共识算法  
  https://docs.chainmaker.org.cn/tech/%E5%85%B1%E8%AF%86%E7%AE%97%E6%B3%95.html
- 长安链文档：数据存储  
  https://docs.chainmaker.org.cn/tech/%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8.html
- 长安链文档：链下扩容技术方案  
  https://docs.chainmaker.org.cn/tech/%E9%93%BE%E4%B8%8B%E6%89%A9%E5%AE%B9%E9%A1%B9%E7%9B%AE%E6%8A%80%E6%9C%AF%E6%96%87%E6%A1%A3.html
- 趣链 FiloInk 司法存证服务平台  
  https://www.hyperchain.cn/en/products/fly
- 趣链 Hyperchain 联盟链底层平台  
  https://www.hyperchain.cn/en/products/hyperchain
- Ethereum：Optimistic Rollups  
  https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/
- Ethereum：ZK Rollups  
  https://ethereum.org/en/developers/docs/scaling/zk-rollups/
- Ethereum：Validium  
  https://ethereum.org/en/developers/docs/scaling/validium/
- 国家标准参考：GB/T 42752-2023《区块链和分布式记账技术 参考架构》、GB/T 43572-2023《区块链和分布式记账技术 术语》、GB/T 43575-2023《区块链和分布式记账技术 系统测试规范》、GB/T 43579-2023《区块链和分布式记账技术 智能合约生命周期管理技术规范》、GB/T 43580-2023《区块链和分布式记账技术 存证通用服务指南》
