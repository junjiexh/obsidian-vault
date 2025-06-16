# 提前写日志

每次操作，比如commit，都会写入一个log
### undo 和 redo log
分为undo log和redo log

`undo log`解决的是原子性的问题，即如果不想写入的被写入了，我们想撤回这些写入。（写入指写入磁盘）
- 场景：STEAL操作

`redo log`解决的是持久性的问题，即 如果想写入的，没有写入，可以重新写入。
- 场景：non-Force操作
	- 如果Force开启，commit之前会将所有页强制写入。
- 场景：写入一半crash了


![[Pasted image 20250610172149.png]]

### 恢复过程

**undo log(Steal/Force)**

记录的是各个record的初始值，这样如果没有commit，可以依次撤销。
例如（FORCE时）
![[Pasted image 20250610173538.png]]
- 特别注意：我们必须保证FLUSH发生在写入undo log之后，不然在flush-写undo之间如果crash，我们无法恢复
- 特别注意：还必须保证FLUSH发生在COMMIT之前（FORCE的语义），不然我们无法判断

**redo log(No Steal/No Force)**

相比与undo log，记录的是更新后的值。写入时点是一致的。

# 加强版：ARIES算法

AREIS：基于语义的恢复与隔离算法(Algorithms for Recovery and Isolation Exploiting Semantics)

## 为什么

undo算法需要查询所有的没有commit的txn，redo算法要找到所有已经commit的txn，耗时太久。

于是我们想用各种机制来降低这里的耗时。

## How

### 核心机制：fuzzy checkpoint

checkPoint会限制一个recovery需要扫描的log数。

为什么是fuzzy，因为simple checkpoint会在写入checkpoint到日志之前，将所有的log写入，将脏页写入，并在此阶段暂停开始新的事务，等待当前所有事务完成。这样太慢！

fuzzy的cp主要是
- 写入check point start
- 不冻结当前状态
- 保存一些元数据（主要是脏页表和事物表的**快照版本**）
- 保存完成后，写入check point end

**脏页表**：
- 保存脏页的recLSN，即第一个造成脏页的log，可以用来恢复脏页

**事务表**：
- 保存active的事务
- 跟踪事务的最新LSN（lastLSN）
### 三个阶段

1. 分析阶段
	- 从脏页表中找出第一个需要redo的log，从那个log开始redo。
2. redo阶段
	- 从第一个需要redo的log，开始回放数据库，写入buffer pool。
3. undo阶段
	- 反向地查找需要undo的log，依次undo。需要维护**toUndo set**

redo是顺序访问的，所以性能还能接受，但是undo如果是根据prevLSN进行回溯，为随机IO。所以我们需要一个更优化的算法。见下面的细节。

### 三个阶段的细节

**redo**

redo从脏页中最小的那个LSN开始，序列地undo。

> [!info]- 不会redo的LSN
> ![[Pasted image 20250615170416.png]]

幂等性：如果redo失败了，再次redo效果是一样的。这是因为，如果之前被redo过了，pageLSN会被更新，下次进行redo时就会跳过这些redo过的log

**undo**

> [!info] - undo过程
> 这里的toUndo是一组需要undo的log的集合，初始时只有事务表中的那些
> ![[Pasted image 20250615172459.png]]

undo的过程因为维护了ToUndo的集合，所以可以全部txn同时处理，而不用对单一txn进行回溯访问。

**读取是反向的，且幂等的**：原因是我们下次再undo时，遇到CLR会跳过（并在必要时重新插入ToUndo或者在日志中插入一个aborted对该txn）。可以将CLR视为一个标记，标识该动作已经做过，让实现变得简单，又不会重复操作。
