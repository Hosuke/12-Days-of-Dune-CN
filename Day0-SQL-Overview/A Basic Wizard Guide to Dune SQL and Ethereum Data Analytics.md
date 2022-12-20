# 欢迎所有魔法师

这里是你在 Dune 中完成所有基本 SQL 魔法师操作所需要知道的所有概念。花时间来学习它们，不要尝试一次性学会所有内容，否则你会让你的大脑短路。

我在本指南中省略了一些 Dune SQL 支持的函数；你可以在这里找到 [基本的 Trino SQL 函数](https://trino.io/docs/current/sql/select.html#having-clause)。

我也会很快发布一份关于优化技巧的高级指南。你需要知道的主要优化提示是，始终在时间上进行过滤，并在唯一 id 或哈希列（如果表中存在）上进行连接。

如果你有任何疑问（对任何函数有疑问或认为我遗漏了一个重要函数），请 [在 Twitter 上私信原作者](https://twitter.com/andrewhong5297) 或 [私信译者](https://twitter.com/geyangh) 告诉我们。

# Dune 中的数据表导航

如果你不知道什么是“表”，可以将每笔交易想象成 Excel 表中的一行。表都有共享/相关的列，最常见的是合约/钱包地址或交易哈希。你可以在这里了解 [关系数据库的基础知识](https://www.digitalocean.com/community/tutorials/understanding-relational-databases)。

Dune 有很多表。如果你已经查看了查询界面中的浏览器，我相信你已经被压倒了。我在下面列出了表的层次结构，这样你就可以理解你看到的图标和它们对应的表类型：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0e3ce0b5-cefd-4d1f-a561-374ec521bd7a/Untitled.png)

最简单的表是“交易”表，其中 `from` 是签署交易的钱包，`to` 是与之交互的地址，而 `input` 则是传递给调用函数的数据。关系需要更多的 EVM 知识来导航，因为你进入了更复杂的表，如 traces/logs 或解码表。交易哈希（`hash` 或 `tx_hash`）在交易表中是唯一的，但在其他表中可能会因为交易的执行方式而出现重复。

所有这些表都以某种方式彼此关联。所有日志和跟踪都可以与交易相关联，所有交易都可以与块相关联。**你只需要专注于理解数据创建的顺序（在链上），以及这如何影响下面表格的填充。**

```
💡 每条链都有自己独立的原始和解码表，一些链会共享 spellbook 表。但无论数据来自哪条链，你都可以跨表查询。

钱包地址（EOA）可以跨链连接，但将 Ethereum 与 Optimism 数据按交易哈希或块号连接是无意义的。

```

1. **原始表：** 这些是 `transactions`、`traces`、`logs` 和 `blocks` 表，它们代表了区块链数据的最原始形式：主要是字节数组。要了解数据流动方式：

    1. 你提交一笔**交易**（调用合约、发送 ETH）
    2. 该交易将引发**跟踪**（合约调用其他合约、部署合约、向某处发送 ETH 等）
    3. 这些函数调用通常在执行时会发出事件（存储在**日志**中）
    4. 每笔交易都在特定的**块**中发生。

```
💡 `creation_traces` 是 traces 的子集，仅跟踪合约部署。

```

```
📕 你可以在这篇文章中阅读关于交易中所有数据如何分解的内容：[SQL on Ethereum: How to Work with All the Data from a Transaction](https://towardsdatascience.com/sql-on-ethereum-how-to-work-with-all-the-data-from-a-transaction-103f94f902e5)。

```

2. **解码表：** 基于提交到合约表（即 `ethereum.contracts`）的合约 ABI，函数和事件被转换为字节签名（`ethereum.signatures`），然后与 `traces` 和 `logs` 进行匹配，从而得到解码表，例如 `uniswap_v2_ethereum.Pair_evt_Swap`，其存储由 pair 工厂创建的所有 pair 合约的所有交换（您可以在事件的 `contract_address` 表中过滤特定的一个）。
   1. **每个函数和事件都有自己的表。** 可以查询的读取函数会出现，但无法查询（例如，`balanceOf` 类似的表将为空）
   2. 所有解码表都带有主要的交易元数据列（如 tx_hash、block_time 和 block_number），并带有 `call` 或 `evt` 前缀。
3. **Spellbook 表：** [这些表是使用基于 SQL 的方法创建的](https://github.com/duneanalytics/spellbook)，基于原始表、解码表或种子文件表。它们在调度程序上运行，因此与原始表/解码表相比，它们具有数据延迟。
 