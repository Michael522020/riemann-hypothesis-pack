# riemann-hypothesis-pack

> **一份可校验的、压缩分发的黎曼猜想 Lean 证明源码包。**

本仓库不包含代码，只包含**构建产物**：把上游 [`leanprover-community/riemann-hypothesis`](https://github.com/leanprover-community/riemann-hypothesis) 仓库 `main` 分支的某个固定提交，原样打包、压缩，并附上完整的校验链。

发布在 **Releases** 里。请从 Release 页面下载，不要 clone 本仓库。

---

## 这个包是什么

| 资产 | 内容 | 原始大小 |
|---|---|---|
| `rh-lean-src-<sha>.tar.zst` | **证明源码本体**（`src/` 目录），zstd -19 | 1,097,434,609 B → 128,500,935 B（**8.54×**） |
| `rh-lean-src-<sha>.tar.xz` | 同上内容，xz -9（更小，解压更慢） | 1,097,434,609 B → 120,232,252 B（**9.13×**） |
| `rh-lean-repo-<sha>.tar.zst` | **完整仓库树**（源码 + CI 脚本 + 比较器 + 验证记录），zstd -19 | — → 128,810,855 B |
| `files-sha256sums.txt.gz` | `src/` 下每个 `.lean` 的 sha256（60,493 条） | 3,209,982 B |
| `SHA256SUMS` | 上述资产的 sha256 | 375 B |

`<sha>` 是被打包的上游提交短哈希，例如 `26c37b40`。**完整 sha 在 Release 说明和 `SHA256SUMS` 旁边。**

---

## 为什么需要它

上游仓库**不发 release、也没有 tag**（2026-11-24 实测：0 个 release / 0 个 tag）。想拿到一份完整的证明源码，通常只有两条路：

1. `git clone` 整个仓库 —— 连同全部历史约 953 MiB；
2. 从 GitHub 的 codeload 下载 tar.gz —— 但它是**浮动的**，`main` 一变内容就变，你无法证明自己拿到的是哪一份。

这个包解决的是第 2 条的问题：**把内容钉死在一个提交上，并给出可复核的校验链。**

同时它比官方归档小得多。同一份内容、同一套参数下的实测对比（详见配套笔记 `17-形式化验证与AI数学证明/02-压缩分发与供应链校验.md`）：

| 算法 | 原版提交（175,187 文件） | 当前 main（60,493 文件） |
|---|---:|---:|
| gzip -9（官方归档用的就是 gzip） | 235,815,509 B | — |
| zstd -19 | 157,670,738 B（比 gzip 小 **33.1%**） | 128,500,935 B |
| xz -9 | 144,258,320 B | 120,232,252 B |

---

## 怎么验证

### 第 1 层：验证下载到的文件

```bash
sha256sum -c SHA256SUMS
```

### 第 2 层：验证解包后的每一个文件

```bash
zstd -d rh-lean-src-<sha>.tar.zst        # 或 xz -d
tar -xf rh-lean-src-<sha>.tar.zst
zcat files-sha256sums.txt.gz > files-sha256sums.txt
sha256sum -c files-sha256sums.txt        # 60,493 行，全部 OK
```

### 第 3 层：验证它对应上游的哪个提交

`files-sha256sums.txt` 里的路径是相对仓库根的 `src/...`。要确认内容与上游一致：

```bash
git clone --filter=blob:none https://github.com/leanprover-community/riemann-hypothesis
cd riemann-hypothesis
git checkout <完整 sha>
# 然后逐文件比对，或直接比对 payload 哈希（见下）
```

**payload 哈希**（对「按路径排序的逐文件 sha256 清单」再取一次 sha256）是一个可以把「整份源码」压缩成一个值的锚点：

| 提交 | payload sha256 |
|---|---|
| `26c37b400bc9cfd43c81d798aa5e798af5c55b94`（当前 main） | `67bc88d6529acb7133cb239e9394604befe6d20e0974bc03894d1407e64f917e` |
| `40e56bab026552e089d7afcfeff08d2defb9c005`（原版提交） | `b59c8a0bad6b814c87b8e44682664313514b1c1cfb619b3191a3aedde0d6179a` |

payload 哈希的算法：

```bash
# 从源码树现算
find src -name '*.lean' -type f -print0 | sort -z | xargs -0 sha256sum | sha256sum

# 从本仓库发布的清单复算（清单本身就已经是这个顺序）
zcat files-sha256sums.txt.gz | sha256sum
```

> ⚠️ **不要对清单再排序一次。** 清单按**路径**排序（`sort -z` 的顺序），不是按哈希排序。按行再排一次会得到 `167d6e3e8ddca19b0a72eff5b4019fab0b6cf810aaeae28c5330144f1d7fcbc7` —— 一个看起来同样合法、但不一样的值。这是「顺序敏感的哈希」最典型的翻车方式。

---

## 打包是**可复现**的

同样的输入 + 同样的命令 = 逐字节相同的输出。关键在这几个 tar 参数：

| 参数 | 作用 |
|---|---|
| `--sort=name` | 目录项按名字排序，不依赖文件系统顺序 |
| `--mtime="@<epoch>"` | 固定时间戳，不留打包时刻 |
| `--owner=0 --group=0 --numeric-owner` | 不记录打包者的用户名/组名 |
| `--format=gnu` | 固定 tar 格式（不同格式的 header 布局不同） |
| `gzip -n` | 不把文件名和时间写进 gzip 头 |

复现命令（`<epoch>` 取上游该提交的 committer 时间）：

```bash
tar --sort=name --format=gnu --owner=0 --group=0 --numeric-owner \
    --mtime="@<epoch>" -cf - src \
  | zstd -19 -T0 -q -o rh-lean-src-<sha>.tar.zst
```

---

## ⚠️ 许可证与来源

**这份证明源码没有许可证。** 上游 README 写得很明确：

> The submission belongs to its author and was published without a license.

也就是说：

- 著作权属于**原作者**（Levent Alpöge），本仓库**不主张任何权利**；
- 本仓库做的是**机械压缩后原样再分发**一个**本来就公开**的内容，压缩不改变任何字节语义；
- 原始作品的条款**优先于**本仓库的任何说明；用于任何目的前请自行确认授权。

本仓库自己的说明文字使用 MIT 协议；**这个 MIT 只覆盖本仓库的文字，不覆盖 Release 里的 Lean 源码**。

原版归档与公告：

- `alpo.ge/remark.pdf`，sha256 `3fc97784b71ddd44ff8aa0a177e99622a1c61b13cfa843a0989ee4a7a0cfeb8d`
- `alpo.ge/lean.tar.gz.part01` … `part14`，拼接后 239,275,554 字节，sha256 `9631b4f020199e580ba86178cb56c6e9b09f690c2369a80dc4334aa93f49c7fc`

---

## 关联阅读

- 详细的技术分析（压缩数据、校验链设计、打包踩坑）：[`Michael522020/ai-study-notes`](https://github.com/Michael522020/ai-study-notes) 的 `17-形式化验证与AI数学证明` 主线
- 上游仓库：[`leanprover-community/riemann-hypothesis`](https://github.com/leanprover-community/riemann-hypothesis)
- 上游验证记录：`verification/NOTES.md`（`https://github.com/leanprover-community/riemann-hypothesis`）

---

## License

本仓库的**说明文字**：MIT。
Release 中的 **Lean 源码**：版权归原作者所有，本仓库不主张任何授权，详见上文「许可证与来源」。
