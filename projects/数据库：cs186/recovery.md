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