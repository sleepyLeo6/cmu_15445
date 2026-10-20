# intro 

这节课的核心不只是理解哈希表如何实现，而是回答三个数据库系统问题：
1. 为什么 DBMS 需要 Hash Table？
2. 当 Hash Collision（哈希冲突）发生时，DBMS 如何高效地处理？
3. 当存储的数据量不断增长时，Hash Table 如何动态扩容，而不必重建整个数据结构？


**Part 1：Background & Hash Functions**<br>
Pages 3–17 · <br>
哈希表在 DBMS 中的作用、时间复杂度、Hash Function 和 Hashing Scheme。

**Part 2：Linear Probe Hashing**<br>
Pages 18–65 · <br>
线性探测、碰撞处理、Load Factor、Tombstone、键值存储与非唯一键。

**Part 3：Cuckoo Hashing**<br>
Pages 66–80 · <br>
多哈希函数、元素驱逐、重新定位，以及为什么查询能保持固定数量的位置检查。

**Part 4：Chained Hashing**<br>
Pages 81–95 · <br>
Bucket 链表、Overflow Bucket，以及利用 Filter 优化查询。

**Part 5：Extendible Hashing**<br>
Pages 96–119 · <br>
Global Depth、Local Depth、Directory 和 Bucket Split。这是本讲的重要难点。

**Part 6：Linear Hashing**<br>
Pages 120–149 · <br>
Split Pointer、渐进式扩容、多哈希函数切换和删除后的收缩。

**Part 7：Summary & Comparison**<br>
Pages 150–152 · <br>
比较各种 Hashing Scheme，并解释为什么一般的数据库索引更倾向于使用 B+ Tree。

# part 1 background & hash functions

![hash tables](./img/07_access_methods.png)