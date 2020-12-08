# bcache

## 代码

### bkey

- 定义

	- u64 high

		- inode 0, 20
		- size  20, 16
		- dirty 36, 1
		- pinned 55, 1
		- csum 56, 2
		- header_size 58, 2 （好像没用）
		- ptrs 60, 3
		- unused 63, 1

	- u64 low

		- offset 0, 64

			- 大部分情况值得是end。
offset in backing dev
    end of the extend

	- u64 ptr[]

		- gen 0, 8
		- offset 8, 43

			- offset in cache device, end of the extend

		- dev 51, 12

- 使用

	- field

		- offset

			- bch_data_insert_start: 
    SET_KEY_OFFSET(k, bio->bi_iter.bi_sector);
			- bch_alloc_sectors:
    SET_KEY_OFFSET(k, KEY_OFFSET(k) + sectors);

		- ptr

			-   94 #define MAKE_PTR(gen, offset, dev)                                      \
  95         ((((__u64) dev) << 51) | ((__u64) offset) << 8 | gen)

		- high

			- [SET_]KEY_SIZE

				- bch_alloc_sectors: 
    SET_KEY_SIZE(k, sectors);

			- #define KEY_START(k)                    (KEY_OFFSET(k) - KEY_SIZE(k))
			- #define START_KEY(k)                    KEY(KEY_INODE(k), KEY_START(k), 0)

		- 创建key

			- KEY（inode, offset, size）

				-   71 #define KEY(inode, offset, size)                                        \
  72 ((struct bkey) {                                                        \
  73         .high = (1ULL << 63) | ((__u64) (size) << 20) | (inode),        \
  74         .low = (offset)                                                 \
  75 })

	- utils

		- bkey_u64s （bkey 包含多少个u64）
		- bkey_bytes (bkey 包含多少个bit = bkey_u64s * sizeof(u64))
		- bkey_idx(struct bkey *k, unsigned int nr_keys)  (nr_keys = keys_u64s, 是通过bkey_u64s 计算出来的，所以直接偏移就可以了)

			-  124 static inline struct bkey *bkey_idx(const struct bkey *k, unsigned int nr_keys)
 125 {
 126         __u64 *d = (void *) k;
 127 
 128         return (struct bkey *) (d + nr_keys);
 129 }

		- bkey_next (下一个bkey)

			-  117 static inline struct bkey *bkey_next(const struct bkey *k)
 118 {
 119         __u64 *d = (void *) k;
 120 
 121         return (struct bkey *) (d + bkey_u64s(k));
 122 }

		- BKEY_PADDED (预留6个ptr的bkey)

			-  130 /* Enough for a key with 6 pointers */
 131 #define BKEY_PAD                8
 132 
 133 #define BKEY_PADDED(key)                                        \
 134         union { struct bkey key; __u64 key ## _pad[BKEY_PAD]; }

### bucket

- atomic_t pin

	- 不需要gc和invalidate，比如在free_inc里面的bucket

		- __bch_invalidate_one_bucket:
    atomic_inc(&b->pin);
		- Subtopic 2

- uint16_t prio

	- for lru

		-  751 #define BTREE_PRIO              USHRT_MAX
 752 #define INITIAL_PRIO            32768U

- uint8_t gen

	- for invalidate

- uint8_t last_gc

	- for invalidate

- uint16_t gc_mark

	- 定义

		- GC_MARK 0, 2

			- GC_MARK_RECLAIMABLE 1
			- GC_MARK_DIRTY 2
			- GC_MARK_METADATA 3

		- GC_SECTORS_USED 2, 13
		- GC_MOVE 15, 1

	- 使用

		- SET_GC_MARK

			- gc

				- btree_gc_start:
所有的bucket，如果pin等于0，设置gc_mark为0，并且设置sectors used 0.
				- bch_btree_gc_root，
遍历整个btree，通过key来mark 对应的bucket。
				- bch_btree_gc_finish

					- uuid_bucket 全都标记成metadata
					- 正在writeback的bucket，标记成dirty
					- sb bucket -》metadata
					- prio bucket -》 metadata

			- allocation

				- bch_bucket_free 
释放bucket的时候mark 成0
				- bch_bucket_alloc
分配bucket的时候，会根据用途mark，data-》reclaimable，metadata-》metadata

			- request

				-  232                 if (op->writeback) {
 233                         SET_KEY_DIRTY(k, true);
 234 
 235                         for (i = 0; i < KEY_PTRS(k); i++)
 236                                 SET_GC_MARK(PTR_BUCKET(op->c, k, i),
 237                                             GC_MARK_DIRTY);
 238                 }

			- register

				- bch_btree_check(c)
 在register_cache 的时候，需要check 所有的bkey。同时会mark bkey 对应的bucket的gc_mark

### utils

- fifo

	- 定义

		-  116 #define DECLARE_FIFO(type, name)                                        \
 117         struct {                                                        \
 118                 size_t front, back, size, mask;                         \
 119                 type *data;                                             \
 120         } name

			- front (where to pop)
			- back (where to push)
			- size （fifo 大小，最多放入这么多个element）
			- mask  （size - 1）
			- type * data (自定义类型的一组内存空间)

	- 操作

		- init_fifio

			-  148 #define init_fifo(fifo, _size, gfp)                                     \
 149 ({                                                                      \
 150         (fifo)->size = (_size);                                         \
 151         if ((fifo)->size > 4)                                           \
 152                 (fifo)->size = roundup_pow_of_two((fifo)->size) - 1;    \
 153         __init_fifo(fifo, gfp);                                         \
 154 })

				-  127 #define __init_fifo(fifo, gfp)                                          \
 128 ({                                                                      \
 129         size_t _allocated_size, _bytes;                                 \
 130         BUG_ON(!(fifo)->size);                                          \
 131                                                                         \
 132         _allocated_size = roundup_pow_of_two((fifo)->size + 1);         \
 133         _bytes = _allocated_size * sizeof(*(fifo)->data);               \
 134                                                                         \
 135         (fifo)->mask = _allocated_size - 1;                             \
 136         (fifo)->front = (fifo)->back = 0;                               \
 137                                                                         \
 138         (fifo)->data = kvmalloc(_bytes, (gfp) & GFP_KERNEL);            \
 139         (fifo)->data;                                                   \
 140 })

		- #define fifo_push(fifo, i)      fifo_push_back(fifo, (i))

			-  174 #define fifo_push_back(fifo, i)                                         \
 175 ({                                                                      \
 176         bool _r = !fifo_full((fifo));                                   \
 177         if (_r) {                                                       \
 178                 (fifo)->data[(fifo)->back++] = (i);                     \
 179                 (fifo)->back &= (fifo)->mask;                           \
 180         }                                                               \
 181         _r;                                                             \
 182 })

		- #define fifo_pop(fifo, i)       fifo_pop_front(fifo, (i))

			-  194 #define fifo_push_front(fifo, i)                                        \
 195 ({                                                                      \
 196         bool _r = !fifo_full((fifo));                                   \
 197         if (_r) {                                                       \
 198                 --(fifo)->front;                                        \
 199                 (fifo)->front &= (fifo)->mask;                          \
 200                 (fifo)->data[(fifo)->front] = (i);                      \
 201         }                                                               \
 202         _r;                                                             \
 203 })

- heap

	- structure

		-   36 #define DECLARE_HEAP(type, name)                                        \
  37         struct {                                                        \
  38                 size_t size, used;                                      \
  39                 type *data;                                             \
  40         } name

			- size
			- used
			- type* data

	- operation

		- heap_swap

			- swap((h)->data[i], (h)->data[j])

		- heap_peek

			-  #define heap_peek(h)    ((h)->used ? (h)->data[0] : NULL)

		- heap_sift （从实现来看应该叫做heap_sift_down）

			-    60 #define heap_sift(h, i, cmp)                                            \
  61 do {                                                                    \
  62         size_t _r, _j = i;                                              \
  63                                                                         \
  64         for (; _j * 2 + 1 < (h)->used; _j = _r) {                       \
  65                 _r = _j * 2 + 1;                                        \
  66                 if (_r + 1 < (h)->used &&                               \
  67                     cmp((h)->data[_r], (h)->data[_r + 1]))              \
  68                         _r++;                                           \
  69                                                                         \
  70                 if (cmp((h)->data[_r], (h)->data[_j]))                  \
  71                         break;                                          \
  72                 heap_swap(h, _r, _j);                                   \
  73         }                                                               \
  74 } while (0)

		- heap_sift_down （实现来看，应该叫做heap_sift[_up]）

			-   76 #define heap_sift_down(h, i, cmp)                                       \
  77 do {                                                                    \
  78         while (i) {                                                     \
  79                 size_t p = (i - 1) / 2;                                 \
  80                 if (cmp((h)->data[i], (h)->data[p]))                    \
  81                         break;                                          \
  82                 heap_swap(h, i, p);                                     \
  83                 i = p;                                                  \
  84         }                                                               \
  85 } while (0)

		- heap_add

			-   87 #define heap_add(h, d, cmp)                                             \
  88 ({                                                                      \
  89         bool _r = !heap_full(h);                                        \
  90         if (_r) {                                                       \
  91                 size_t _i = (h)->used++;                                \
  92                 (h)->data[_i] = d;                                      \
  93                                                                         \
  94                 heap_sift_down(h, _i, cmp);                             \
  95                 heap_sift(h, _i, cmp);  (不需要)                                \
  96         }                                                               \
  97         _r;                                                             \
  98 })
  99 

		- heap_pop

			-  100 #define heap_pop(h, d, cmp)                                             \
 101 ({                                                                      \
 102         bool _r = (h)->used;                                            \
 103         if (_r) {                                                       \
 104                 (d) = (h)->data[0];                                     \
 105                 (h)->used--;                                            \
 106                 heap_swap(h, 0, (h)->used);                             \
 107                 heap_sift(h, 0, cmp);                                   \
 108         }                                                               \
 109         _r;                                                             \
 110 })

- closure

### journal

- data structure

	- journal_replay

		- list （从journal_bucket读出数据，生成journal_replay 并且挂载到一个list里面，后面通过这个list，遍历所有的journal_replay，然后回放）
		- pin
		- jset

	- jset (journal 的磁盘结构，包含新的key和一些统计信息，比如csum)

		- csum

			- csum_set(w->data);

		- magic

			- sb->set_magic ^ JSET_MAGIC;

		- version (1)
		- last_seq （最老一个还不能释放的jset）

			- ((j)->seq - fifo_used(&(j)->pin) + 1)

		- seq (一个一直增长的序号)
		- keys （这个jset里面的bkey数量）
		- btree_root （当前的btree 根节点的bkey）
		- uuid_bucket （当前uuid bucket的bkey）
		- btree_level （当前btree的层数，也就是root的btree level）
		- prio_bucket （第一个priority bucket）
		- start （jset里面的第一个bkey）

	- journal

		- 和cache关联的数据结构，每个cache有一个journal，是嵌在cache_set数据结构里面的一个成员。

			- flush_write_lock; 这个用来保护c->journal.btree_flushing
			- btree_flushing，避免重复的调用btree_flush_write
			- blocks_free， 当前bucket剩余的空闲block数。在journal_write的时候回减少。在insert journal的时候回检查。
			- seq 当前jset的seq，每一个jset唯一的seq
			- DECLARE_FIFO(atomic_t, pin); 当前open的journal的记录。

	- journal_device

		- seq[SB_JOURNAL_BUCKETS]; 所有journal bucket包含的jset的最大seq。

			- seq是journal 数据结构里面的seq不断增长的结果，在bch_journal_next 当中会增加1， 并且赋给下一个jset；
			- 在journal_write里面，会更新journal_device->seq[bucket_id]成为当前要写入的jset的seq，这个seq一定是最大的，因为seq是单向增长的。
			- seq 的目的是通过他记录的最大seq来判断这个bucket是否还包含open的journal。如果seq < journal->last_seq. 就说明这个bucket可以被回收了。

		- cur_idx

			- 正在写入journal的bucket
			- 在journal_reclaim 会更新
			- 如果发现bucket 当前没有空间了，会新分配下一个bucket。
unsigned int next = (ja->cur_idx + 1) % ca->sb.njournal_buckets

		- last_idx

			- 最老的bucket 包含open journal的bucket。
			- 在reclaim的时候更新，如果seq 小于last_seq，更新last_idx。

		- discard_idx
		- bio

			- 缓存盘的journal 读写都用这个bio

- operation

	- bch_data_insert_keys 

（先写数据，然后把key插入到journal，最后吧key插入btree）

		- journal_ref = bch_journal()

			- w = journal.cur (找到当前正在使用的 journal_write，如果满了，write journal，然后使用另一个journal_write)
			- memcpy(bset_bkey_last(w->data), keys->keys, bch_keylist_bytes(keys));
			- w->data->keys += bch_keylist_nkeys(keys); （增加jset 的大小）
			- ret = &fifo_back(&c->journal.pin);
atomic_inc(ret);  （拿到一个ref， 会在稍后释放）

		-   ret = bch_btree_insert(op->c, &op->insert_keys,
                                 journal_ref, replace_key)


			- （将key 插入到btree，传入了journal_ref，当插入成功了之后，并且这个journal是当前btree里面的最老journal，会设置btree_write->journal, 并且atomic_inc(journal_ref) （bch_btree_leaf_dirty）. 也就是说，btree 拿到一个journal ref， 
			- 当btree 写到磁盘之后，会释放这个ref （btree_complete_write））
			- replace 用于吧cache miss的数据插入到cache，replace的key不需要journal。

		-            if (journal_ref)
                   atomic_dec_bug(journal_ref);

 

			-  释放上面拿到的ref， 不是释放btree 拿到的ref

	- journal_write

		- 减小blocks_free，一个bucket，可能有很多jset。
		- 构造jset，包括 btree_level，uuid_bucket，btree_root，prio_bucket。magic，version, last_seq, csum.

			- 有些数据应该放到superblock里面，但是由于更改频繁，只需要记录到jset里面就行了。重新启动的时候读取最新的jset获取这些信息。

		- bch_bio_map(bio, w->data);

			- 映射bio 数据

		- atomic_dec_bug(&fifo_back(&c->journal.pin));

			- 减少pin，如果journal里面的key都已经落盘，可能直接变成0，表示可以被reclaim

		- bch_journal_next(&c->journal);

			- fifo_push(&j->pin, p) 在j->pin里面增加一个元素，表示一个新的journal被打开。
			- atomic_set(&fifo_back(&j->pin), 1); 初始化为1， 写入的时候回减掉。 每一个没有罗盘的key会拿一个ref。
			- j->cur->data->seq       = ++j->seq;

		- journal_reclaim(c);
		-  while ((bio = bio_list_pop(&list)))
             closure_bio_submit(c, bio, cl);

			- 提交write bio，但是这个地方有可能已经被discard，在去写就需要重新申请空间。

	- bch_journal_read

		- DECLARE_BITMAP(bitmap, SB_JOURNAL_BUCKETS);
bitmap_zero(bitmap, SB_JOURNAL_BUCKETS);

			- 使用一个bitmap来记录所有journal_bucket是否已读。

		-  195                 /*
 196                  * Read journal buckets ordered by golden ratio hash to quickly
 197                  * find a sequence of buckets with valid journal entries
 198                  */
 199                 for (i = 0; i < ca->sb.njournal_buckets; i++) {
 200                         /*
 201                          * We must try the index l with ZERO first for
 202                          * correctness due to the scenario that the journal
 203                          * bucket is circular buffer which might have wrapped
 204                          */
 205                         l = (i * 2654435769U) % ca->sb.njournal_buckets;
 206 
 207                         if (test_bit(l, bitmap))
 208                                 break;
 209 
 210                         if (read_bucket(l))
 211                                 goto bsearch;
 212                 }

			- 使用黄金分割点来跳跃式查找第一个有合法journal entry的bucket。

有两种bucket 是需要跳过的。
（1） 没有journal的bucket
（2）开头不是完整的jset(这种情况现在已经不存在)

				- 由于现在jset没有跨bucket的场景，这部分逻辑可以简化？

		-  214                 /*
 215                  * If that fails, check all the buckets we haven't checked
 216                  * already
 217                  */
 218                 pr_debug("falling back to linear search\n");
 219 
 220                 for_each_clear_bit(l, bitmap, ca->sb.njournal_buckets)
 221                         if (read_bucket(l))
 222                                 goto bsearch;
 223 
 224                 /* no journal entries on this device? */
 225                 if (l == ca->sb.njournal_buckets)
 226                         continue;

			- 如果黄金分割找不到合法的journal，吧剩下的bucket 顺序扫一遍。如果还没有，说明没有journal可用。

		-  227 bsearch:
 228                 BUG_ON(list_empty(list));
 229 
 230                 /* Binary search */
 231                 m = l;
 232                 r = find_next_bit(bitmap, ca->sb.njournal_buckets, l + 1);
 233                 pr_debug("starting binary search, l %u r %u\n", l, r);
 234 
 235                 while (l + 1 < r) {
 236                         seq = list_entry(list->prev, struct journal_replay,
 237                                          list)->j.seq;
 238 
 239                         m = (l + r) >> 1;
 240                         read_bucket(m);
 241 
 242                         if (seq != list_entry(list->prev, struct journal_replay,
 243                                               list)->j.seq)
 244                                 l = m;
 245                         else
 246                                 r = m;
 247                 }
 248 

			- 如果找到第一个合法的bucket，下一步目的是找到seq 最大的jset，也就是最新的jset。

				- 首先确定最大seq jset的范围。

					- 如果第一个bucket 合法

						- 如果journal 溢出，回到头一个bucket，第一次就会发现第一个bucket是合法的，那么范围就是0- max也没问题。
						- journal 没有溢出，第一个bucket就是最老的jset，范围是0-max，没问题。

					- 如果第一个bucket 不合法，

						- 说明journal 这一轮一定没有出现回头的情况，就是连续的一段bucket。那么找到一个合法的bucket：n，然后范围就是n-max

				- 先拿到当前list里面的最后一个jset
				- 读出范围中的中间值，m = (l + r ) / 2
				- 再次拿到list最后一个jset的seq

					- 如果变化了，说明m 的seq 比较大，最大值应该在 m - r

						- 如果journal是连续的，这个逻辑很自然
						- 如果journal回头了，我们的l 一定回头的尾部，因为回头的场景，第一个bucket一定是合法的，l初始值一定是0.
这种情况最后一个jset也是在 m - r 也没问题。

					- 如果没有变化，说明m 要么不合法，要么seq 比较小。这个时候 最大值应该在 l - m

						- m 不合法，说明m已经超过最后一个jset了。
						- 如果m 比较小，有可能发生在journal 回头，但是m 到了journal的头部。这时候也是同样的 最大值在l - m

				- 直到最后找到最大的seq jset

		-  249                 /*
 250                  * Read buckets in reverse order until we stop finding more
 251                  * journal entries
 252                  */
 253                 pr_debug("finishing up: m %u njournal_buckets %u\n",
 254                          m, ca->sb.njournal_buckets);
 255                 l = m;
 256 
 257                 while (1) {
 258                         if (!l--)
 259                                 l = ca->sb.njournal_buckets - 1;
 260 
 261                         if (l == m)
 262                                 break;
 263 
 264                         if (test_bit(l, bitmap))
 265                                 continue;
 266 
 267                         if (!read_bucket(l))
 268                                 break;
 269                 }

			- 从最大值逆向读出所有的bucket

		-  271                 seq = 0;
 272 
 273                 for (i = 0; i < ca->sb.njournal_buckets; i++)
 274                         if (ja->seq[i] > seq) {
 275                                 seq = ja->seq[i];
 276                                 /*
 277                                  * When journal_reclaim() goes to allocate for
 278                                  * the first time, it'll use the bucket after
 279                                  * ja->cur_idx
 280                                  */
 281                                 ja->cur_idx = i;
 282                                 ja->last_idx = ja->discard_idx = (i + 1) %
 283                                         ca->sb.njournal_buckets;
 284 
 285                         }

			- 设置当前seq以及idx

	- bch_journal_replay

		- 计算start 和end

			- start 通过最新的jset里面的last_seq得到
			- end 通过最新的jset的seq得到

		- 遍历list，并且一一insert到btree

			- 如果jset的seq 小于start，这种情况不可能发生，在journal_read的时候，已经删除了所有小于last_seq的jset
			- 如果第一个jset的seq 大于last_seq。这种情况有可能发生：

journal_write里面，首先释放了jset的pin，然后调用了journal_reclaim，如果启用了discard，会discard调对应的bucket，最后再去写入jset。

有一种可能在reclaim之后，已经discard了bucket，但是最新的jset还没有写入。

导致replay的时候看到last_seq 开始的一部分jset丢了。这种情况会打印一个info，但是没有关系，因为能够被discard的jset，里面的bkey都一定已经写入到btree。

	- journal_reclaim

		-  while (!atomic_read(&fifo_front(&c->journal.pin)))
                 fifo_pop(&c->journal.pin, p);

			- pop掉所有已经为0的journal，如果seq 对应的pin为0，说明没有人在饮用这个journal，可以被回收。

		- last_seq = last_seq(&c->journal);

			- 计算最老的journal，每一个journal 在打开的时候（bch_journal_next） 

		- 使用新的last_seq，计算新的last_idx
		- 如果有必要，discard不需要的journal bucket

			- 这里没有判断journal_write是否结束，理论上存在一种可能，discard之后，write有重新分配了新的数据。这样期望回收的空间没有达到回收效果。

		- 判断blocks_free，如果不够，新分配bucket。

			- journal_write的时候回减去写入的容量。
			- bch_data_insert_keys的时候不会减少，但是判断的时候，会通过journal 总大小来判断，也就是说，需要写入jset的总大小来和blocks_free比较。

	- btree_flush_write 触发btree写入，从最老的journal对应的bkey开始写，目的是为了尽快释放journal，以回收journal空间。

		- 找到引用最老的journal的btree，flush。

### bset

- data structure

	- bset

		- csum 写入的时候计算csum
		- magic

			-  static inline __u64 bset_magic(struct cache_sb *sb)
 {
         return sb->set_magic ^ BSET_MAGIC;
 }

		- seq

			- 同一个btree 下面的所有bset都拥有同一个seq，这是一个随机值，新的btree会获得一个新的随机值。

		- version

			- #define BCACHE_BSET_VERSION             1

		- keys

			- bch_bset_insert 函数里面会更新：
t->data->keys += bkey_u64s(insert);

		- start

	- btree_keys
	- btree_iter

		- 遍历btree的迭代器

			- definition
定义其实是一个heap，可以参考一下heap的定义，这就是一个type 是btree_iter_set的heap

				- size_t size, used;

					- heap的size 和used。

				- struct btree_iter_set {
     struct bkey *k, *end;
 }
data [MAX_BSETS]

					- heap的type，就是btree_iter_set，他表示一段连续的bkey，从k到end。

			- operations

				- init

					- __bch_btree_iter_init

						- __bch_btree_iter_init

				- Subtopic 2

	- bset_tree
	- btree_keys_ops
	- key_list

		- definition

			-  486         union {
 487                 struct bkey             *keys;
 488                 uint64_t                *keys_p;
 489         };

				- keys 表示所有keylist的key 所在的区域，可能是inline，也可能是kmalloc分配的。默认指向inline

			-  490         union {
 491                 struct bkey             *top;
 492                 uint64_t                *top_p;
 493         };

				- top 表示keylist的头，默认指向keys，push就会往后移动，pop就会往前移动，pop_front删除第一个元素，top还是指向keylist最后。

			-  495         /* Enough room for btree_split's keys without realloc */
 496 #define KEYLIST_INLINE          16
 497         uint64_t                inline_keys[KEYLIST_INLINE];

				- 默认inline 给16个uint64位置。

		- operation

			-  500 static inline void bch_keylist_init(struct keylist *l)
 501 {
 502         l->top_p = l->keys_p = l->inline_keys;
 503 }

				- 初始化一个keylist，默认top和keys都指向inline_keys

			-  505 static inline void bch_keylist_init_single(struct keylist *l, struct bkey *k)
 506 {
 507         l->keys = k;
 508         l->top = bkey_next(k);
 509 }

				- 制定一个key用来初始化keylist，keys指向这个k，top指向key的下一个位置。

			- push

				- l->top = bkey_next(l->top);

			- pop

				-  159 struct bkey *bch_keylist_pop(struct keylist *l)
 160 {
 161         struct bkey *k = l->keys;
 162 
 163         if (k == l->top)
 164                 return NULL;
 165 
 166         while (bkey_next(k) != l->top)
 167                 k = bkey_next(k);
 168 
 169         return l->top = k;
 170 }

					- 找到最后一个key的开始位置，然后把top移动到最后一个key的开始位置。

			- pop_front

				- 173 void bch_keylist_pop_front(struct keylist *l)
 174 {
 175         l->top_p -= bkey_u64s(l->keys);
 176 
 177         memmove(l->keys,
 178                 bkey_next(l->keys),
 179                 bch_keylist_bytes(l));
 180 }

					- 删除第一个key。可能存在内存越界访问？

没有越界访问，因为先移动了top_p，也就是说，bch_keylist_bytes里面会使用新的top_p来计算bytes，已经是减掉第一个key之后的bytes。移动的只有需要的数据。

			- bch_keylist_add

				-  518         bkey_copy(l->top, k);
 519         bch_keylist_push(l);

					- 将key copy到top的位置，并且移动top到下一个位置。top总是指向下一个没有数据的位置。

			- bch_keylist_free

				-  534         if (l->keys_p != l->inline_keys)
 535                 kfree(l->keys_p);

					- inline 数据不需要free，只需要free 外部内存。

			-  538 static inline size_t bch_keylist_nkeys(struct keylist *l)
 539 {
 540         return l->top_p - l->keys_p;
 541 }
 542 
 543 static inline size_t bch_keylist_bytes(struct keylist *l)
 544 {
 545         return bch_keylist_nkeys(l) * sizeof(uint64_t);
 546 }

			- realloc

				- int __bch_keylist_realloc(struct keylist *l, unsigned int u64s)

					- 为keylist 增加u64s 个key的位置。

						- 先计算old_size，等于现在keylist里面的key个数。
						- 然后计算new_size，等于old_size 加上新增加的key个数。

还需要做一个round_up
roundup_pow_of_two(newsize);
						- 检查一下newsize，如果小于inline或者newsize等于oldsize，就不需要重新分配，直接返回。
 140         if (newsize <= KEYLIST_INLINE ||
 141             roundup_pow_of_two(oldsize) == newsize)
 142                 return 0;
						- new_keys = krealloc(old_keys, sizeof(uint64_t) * newsize, GFP_NOIO);
						- 如果以前使用的是inline，还需要copy一下inline的数据到new_keys
						- 设置keys为new_keys，设置top为new_keys + old_size

- operations

	- bch_bset_insert （）

		- Subtopic 1

### btree

- data  structure

	- btree

		- struct hlist_node       hash;
		- BKEY_PADDED(key);
		- seq （每次写加锁或者写释放锁，都会增加）
		- lock
		- c cache_Set 的引用
		- parent，父节点
		- write_lock，btree write的时候会用到
		- flags

			- io_error
			- dirty
			- write_idx
			- journal_flush

		- closure io
		- btree_write writes[2] 两个write，一个pending， 一个inflight
		- bio

	- btree_write
	- btree_op
	- btree_check_info
	- btree_check_state
	- gc_merge_info
	- btree_insert_op
	- refill

- operations

	- bch_btree_node_read

		- alloc  bio
		- bch_bio_map（bio, ）

	- btree_write
	- btree cache
	- btree check
	- gc

		- bch_gc_thread

			- set_gc_sectors

				-  195 static inline void set_gc_sectors(struct cache_set *c)
 196 {
 197         atomic_set(&c->sectors_to_gc, c->sb.bucket_size * c->nbuckets / 16);
 198 }

			- bch_btree_gc

				- btree_gc_start

					- c->gc_mark_valid = 0
					- for each bucket,
 b->last_gc = b->gen
					- if (!atomic_read(b->pin))
    SET_GC_MARK(b, 0)
    SET_GC_SECTORS_USED(b, 0)

				- bch_btree_gc_root

					- should_rewrite = btree_gc_mark_node

						- should_rewrite is true

							- btree_node_alloc_replacement
							- bch_btree_node_write_sync()
							- bch_btree_set_root
							- return -EINTR 返回-EINTR之后，会重新开始

					- __bch_btree_mark_key(b->c, b->level + 1, &b->key)

mark 这个bucket 本身的key，注意这里的level + 1， 用来表示这个bucket的key所处于的level，也就是这个bucket的level的上一级。

这个值，用于mark_key函数里面的bug_on。

						- g = PTR_BUCKET(c, k, i)
						- 如果 gen_after(g->last_gc, PTR_GEN(k, i)), 那么设置last_gc，g->last_gc = PTR_GEN(k, i)

找到btree当中指向这个bucket的最老的那个bkey，然后last_gc 就是这个bkey里面的gen。

目的是为了在判断一个bucket是否可以被invalidate。如果一个bucket last_gc 比gen 大了超过128， 表示不能被invalidate。这样的bucket太久没有gc了，需要触发gc了。

							- gen_after(uint8_t a, uint8_t b)

uint8_t r = a - b;
return r > 128 ? 0 : r;

也就是说，如果a 比b 大的程度不超过128， 那么我们就认为gen_after 返回r，超过128，返回0. 注意，小于减下来也是超过128.

						- ptr_stale(c, k, i)
如果k是stale，那么就跳过这个key，并且设置记录下最大的stale值。

stale = max(stale, ptr_stale(c, k, i))
目的是找到当前这个metadata的bucket里面最大的stale值。后面会根据这个bucket里面最大的stale值考虑进行rewrite。
						- 1220                 cache_bug_on(GC_MARK(g) &&
1221                              (GC_MARK(g) == GC_MARK_METADATA) != (level != 0),
1222                              c, "inconsistent ptrs: mark = %llu, level = %i",
1223                              GC_MARK(g), level);

							- 这个bug_on 很绕，简单来说，就是如果bkey指向的bucket是metadata，那么key所在的level不等于0.

						- 1225                 if (level)
1226                         SET_GC_MARK(g, GC_MARK_METADATA);
1227                 else if (KEY_DIRTY(k))
1228                         SET_GC_MARK(g, GC_MARK_DIRTY);
1229                 else if (!GC_MARK(g))
1230                         SET_GC_MARK(g, GC_MARK_RECLAIMABLE);

							- 真正的mark工作，

如果不是leaf，那么指向的bucket就一定是metadata bucket，mark上，不能被invalidate。

如果是leaf，并且key是dirty，那么mark 指向的bucket为dirty，不能被invalidate。

如果是leaf，而且key不是dirty，如果现在的bucket没有被mark过，就mark成为reclaimable。

						- 1232                 /* guard against overflow */
1233                 SET_GC_SECTORS_USED(g, min_t(unsigned int,
1234                                              GC_SECTORS_USED(g) + KEY_SIZE(k),
1235                                              MAX_GC_SECTORS_USED));

							- 将key指向的size加到这个bucket的used里面。

这个值会用在movinggc使用到。

						- return stale

					- 1683         if (b->level) {
1684                 ret = btree_gc_recurse(b, op, writes, gc);
1685                 if (ret)
1686                         return ret;
1687         }

						- 如果不是leaf，那么需要对btree调用
btree_gc_recurse（）迭代下去。

							- bch_btree_iter_init(&b->keys, &iter, &b->c->gc_done);

从已经结束的bkey开始下一次gc
							- 1581         for (i = r; i < r + ARRAY_SIZE(r); i++)
1582                 i->b = ERR_PTR(-EINTR);

r 是一个数组：struct gc_merge_info r[GC_MERGE_NODES];
GC_MERGE_NODES 默认等于4

也就是说，每次可以探测连续的四个node是否可以merge。
							- 1584         while (1) {
1585                 k = bch_btree_iter_next_filter(&iter, &b->keys, bch_ptr_bad);
}

一个循环，不断找到下一个bkey。对每一个bkey进行mark。

								- 1587                         r->b = bch_btree_node_get(b->c, op, k, b->level - 1,
1588                                                   true, b);
1589                         if (IS_ERR(r->b)) {
1590                                 ret = PTR_ERR(r->b);
1591                                 break;
1592                         }
1593 
1594                         r->keys = btree_gc_count_keys(r->b);
1595 
1596                         ret = btree_gc_coalesce(b, op, gc, r);
1597                         if (ret)
1598                                 break;

读出这个bkey指向的bucket，并且调用btree_gc_coalesce（）判断是否可以merge。

									- btree_gc_coalesce（）

								- 1601                 if (!last->b)
1602                         break;

last 是指向r 数据最后一个元素的指针，我们会从last开始处理，处理完成一个之后，就把整个数组往后移动一位，并且设置第一个位置为NULL。（r所有成员初始化为-EINTR）

所以如果发现last已经是NULL，那么r数组就已经空了。直接跳出循环即可。

也就是说r有几种形态如下：
（1）初始化：【-EINTR，-EINTR，-EINTR，-EINTR】
（2）独处第一个数据：【b, -EINTR，-EINTR，-EINTR】这个时候不会开始处理，我们只会从最后一个位置开始处理。
（3）读出来一个继续往后移动：【b, b，-EINTR，-EINTR】,【b, b，b，-EINTR】
（4）直到最后一个位置有数据，那么开始处理：【b, b, b, b】
								- 1604                 if (!IS_ERR(last->b)) {
1605                         should_rewrite = btree_gc_mark_node(last->b, gc);
1606                         if (should_rewrite) {
1607                                 ret = btree_gc_rewrite_node(b, op, last->b);
1608                                 if (ret)
1609                                         break;
1610                         }
1611 
1612                         if (last->b->level) {
1613                                 ret = btree_gc_recurse(last->b, op, writes, gc);
1614                                 if (ret)
1615                                         break;
1616                         }
1617 
1618                         bkey_copy_key(&b->c->gc_done, &last->b->key);
1619 
1620                         /*
1621                          * Must flush leaf nodes before gc ends, since replace
1622                          * operations aren't journalled
1623                          */
1624                         mutex_lock(&last->b->write_lock);
1625                         if (btree_node_dirty(last->b))
1626                                 bch_btree_node_write(last->b, writes);
1627                         mutex_unlock(&last->b->write_lock);
1628                         rw_unlock(true, last->b);
1629                 }
								- 1631                 memmove(r + 1, r, sizeof(r[0]) * (GC_MERGE_NODES - 1));
1632                 r->b = NULL;

r 整体向后移动一位
								- 1634                 if (atomic_read(&b->c->search_inflight) &&
1635                     gc->nodes >= gc->nodes_pre + btree_gc_min_nodes(b->c)) {
1636                         gc->nodes_pre =  gc->nodes;
1637                         ret = -EAGAIN;
1638                         break;
1639                 }

incremental gc，这里会在每次gc完成一个node之后判断一下，这一轮gc完成了多少个node，如果太多，那就返回EAGAIN，下次再来。避免一直gc 影响业务IO。

									- 1543 static size_t btree_gc_min_nodes(struct cache_set *c)
1544 {
1545         size_t min_nodes;
1546 
1547         /*
1548          * Since incremental GC would stop 100ms when front
1549          * side I/O comes, so when there are many btree nodes,
1550          * if GC only processes constant (100) nodes each time,
1551          * GC would last a long time, and the front side I/Os
1552          * would run out of the buckets (since no new bucket
1553          * can be allocated during GC), and be blocked again.
1554          * So GC should not process constant nodes, but varied
1555          * nodes according to the number of btree nodes, which
1556          * realized by dividing GC into constant(100) times,
1557          * so when there are many btree nodes, GC can process
1558          * more nodes each time, otherwise, GC will process less
1559          * nodes each time (but no less than MIN_GC_NODES)
1560          */
1561         min_nodes = c->gc_stats.nodes / MAX_GC_TIMES;
1562         if (min_nodes < MIN_GC_NODES)
1563                 min_nodes = MIN_GC_NODES;
1564 
1565         return min_nodes;
1566 }

两个参数控制每次gc最小需要完成的node数量，一个是MAX_GC_TIMES，100。gc 最多运行100次，看看每次运行多少个。同时，每次运行个数，不能小于MIN_GC_NODES。

如果我们让gc过程中也可以invalidate，其实gc可以时间更长，可以通过增加MAX_GC_TIMES来减小gc 对业务IO的影响。

								- 1641                 if (need_resched()) {
1642                         ret = -EAGAIN;
1643                         break;
1644                 }

							- 1647         for (i = r; i < r + ARRAY_SIZE(r); i++)
1648                 if (!IS_ERR_OR_NULL(i->b)) {
1649                         mutex_lock(&i->b->write_lock);
1650                         if (btree_node_dirty(i->b))
1651                                 bch_btree_node_write(i->b, writes);
1652                         mutex_unlock(&i->b->write_lock);
1653                         rw_unlock(true, i->b);
1654                 }

while 循坏出来之后，需要对r 数组里面所有node 写入一遍。

					- 1689         bkey_copy_key(&b->c->gc_done, &b->key);

						- 设置gc_done 为当前已完成bkey

				- bch_btree_gc_finish

					- 1728         set_gc_sectors(c);
1729         c->gc_mark_valid = 1;
1730         c->need_gc      = 0;
					- 1732         for (i = 0; i < KEY_PTRS(&c->uuid_bucket); i++)
1733                 SET_GC_MARK(PTR_BUCKET(c, &c->uuid_bucket, i),
1734                             GC_MARK_METADATA);

设置uuid 的bucket，mark为metadata。
					- 1736         /* don't reclaim buckets to which writeback keys point */
1737         rcu_read_lock();
1738         for (i = 0; i < c->devices_max_used; i++) {
1739                 struct bcache_device *d = c->devices[i];
1740                 struct cached_dev *dc;
1741                 struct keybuf_key *w, *n;
1742                 unsigned int j;
1743 
1744                 if (!d || UUID_FLASH_ONLY(&c->uuids[i]))
1745                         continue;
1746                 dc = container_of(d, struct cached_dev, disk);
1747 
1748                 spin_lock(&dc->writeback_keys.lock);
1749                 rbtree_postorder_for_each_entry_safe(w, n,
1750                                         &dc->writeback_keys.keys, node)
1751                         for (j = 0; j < KEY_PTRS(&w->key); j++)
1752                                 SET_GC_MARK(PTR_BUCKET(c, &w->key, j),
1753                                             GC_MARK_DIRTY);
1754                 spin_unlock(&dc->writeback_keys.lock);
1755         }
1756         rcu_read_unlock();

正在writeback的所有bucket mark 成dirty，不可以被reclaim。
					- 1758         c->avail_nbuckets = 0;
1759         for_each_cache(ca, c, i) {
1760                 uint64_t *i;
1761 
1762                 ca->invalidate_needs_gc = 0;
1763 
1764                 for (i = ca->sb.d; i < ca->sb.d + ca->sb.keys; i++)
1765                         SET_GC_MARK(ca->buckets + *i, GC_MARK_METADATA);
1766 
1767                 for (i = ca->prio_buckets;
1768                      i < ca->prio_buckets + prio_buckets(ca) * 2; i++)
1769                         SET_GC_MARK(ca->buckets + *i, GC_MARK_METADATA);
1770 
1771                 for_each_bucket(b, ca) {
1772                         c->need_gc      = max(c->need_gc, bucket_gc_gen(b));
1773 
1774                         if (atomic_read(&b->pin))
1775                                 continue;
1776 
1777                         BUG_ON(!GC_MARK(b) && GC_SECTORS_USED(b));
1778 
1779                         if (!GC_MARK(b) || GC_MARK(b) == GC_MARK_RECLAIMABLE)
1780                                 c->avail_nbuckets++;
1781                 }
1782         }

（1） 计算avail_nbuckets，gc之后可以更新cache_available
（2）mark sb的bueckt为METADATA
（3） mark prio的bucket为METADATA
（4） 计算need_gc 为所有bucket里面最大的gc_gen

	- btree_split

		- n1 = btree_node_alloc_replacement(b, op);

先分配一个n1，用来替换以前的btree。

			- struct btree *n = bch_btree_node_alloc(b->c, op, b->level, b->parent);

首先分配一个btree_node，同样的level，同样的parent。

				- if (__bch_bucket_alloc_set(c, RESERVE_BTREE, &k.key, 1, wait))

先在物理磁盘空间上面分配一个bucket给这个btree，也就是说一个btree使用一个bucket？

					- long b = bch_bucket_alloc(ca, reserve, wait); 

在cache盘中分配一个bucket，b
					-  513                 k->ptr[i] = MAKE_PTR(ca->buckets[b].gen,
 514                                 bucket_to_sector(c, b),
 515                                 ca->sb.nr_this_dev);

将对应的ptr 设置为正确的值，也就是知道该cache的对应bucket上面。

				- SET_KEY_SIZE(&k.key, c->btree_pages * PAGE_SECTORS);

					- 设置key的大小为btree_pages，btree_pages是计算出来的，表示一个btree使用的内存量。

所以一个btree bucket 只有btree_pages 这部分可以被使用？

						- meta_bucket_pages(&c->sb)

							-  765 static inline unsigned int meta_bucket_pages(struct cache_sb *sb)
 766 {
 767         unsigned int n, max_pages;
 768 
 769         max_pages = min_t(unsigned int,
 770                           __rounddown_pow_of_two(USHRT_MAX) / PAGE_SECTORS,
 771                           MAX_ORDER_NR_PAGES);
 772 
 773         n = sb->bucket_size / PAGE_SECTORS;
 774         if (n > max_pages)
 775                 n = max_pages;
 776 
 777         return n;
 778 }

				- b = mca_alloc(c, op, &k.key, level);

分配一个内存空间用来存放这个btree

					- mca_find
先找一下在mca 里面是不是已经有了这个bucket对应的btree 内存实例。
					- 如果没有找到btree，就去btree_cache_freeable 链表里面找一下。

这个链表连接了所有从hash链表里面删除的btree，但是所有内存都还在，包括btree本身和btree的data（keys）。

						-  532 static void mca_bucket_free(struct btree *b)
 533 {
 534         BUG_ON(btree_node_dirty(b));
 535 
 536         b->key.ptr[0] = 0;
 537         hlist_del_init_rcu(&b->hash);
 538         list_move(&b->list, &b->c->btree_cache_freeable);
 539 }

					- 如果freeable链表也没有找到，下面去btree_cache_freed链表里面去找，这个链表保存所有从freeable链表过来的btree，区别在于这个链表里面的btree，data都已经free了，但是btree本身还在。复用只需要重新分配data（keys）就行了。

						-  522 static void mca_data_free(struct btree *b)
     /* [previous][next][first][last][top][bottom][index][help]  */
 523 {
 524         BUG_ON(b->io_mutex.count != 1);
 525 
 526         bch_btree_keys_free(&b->keys);
 527 
 528         b->c->btree_cache_used--;
 529         list_move(&b->list, &b->c->btree_cache_freed);
 530 }

					- 如果freeable和freed 都没有找到可以复用的btree，只能重新去分配一个btree

mca_bucket_alloc（）

						- struct btree *b = kzalloc(sizeof(struct btree), gfp);
首先分配btree 内存本身，这个内存分配之后就不会再释放。
						- mca_data_alloc(b, k, gfp); 
分配btree实际使用的key的内存。

分配成功就放入btree_cache链表。

如果分配失败，放入到btree_freed链表。

							-  546 static void mca_data_alloc(struct btree *b, struct bkey *k, gfp_t gfp)
 547 {
 548         if (!bch_btree_keys_alloc(&b->keys,
 549                                   max_t(unsigned int,
 550                                         ilog2(b->c->btree_pages),
 551                                         btree_order(k)),
 552                                   gfp)) {
 553                 b->c->btree_cache_used++;
 554                 list_move(&b->list, &b->c->btree_cache);
 555         } else {
 556                 list_move(&b->list, &b->c->btree_cache_freed);
 557         }
 558 }

			- bch_btree_sort_into(&b->keys, &n->keys, &b->c->sort);

				- bch_btree_iter_init(b, &iter, NULL);
				- btree_mergesort(b, new->set->data, &iter, false, true);

			- bkey_copy_key(&n->key, &b->key);

				- SET_KEY_INODE(dest, KEY_INODE(src));
				- SET_KEY_OFFSET(dest, KEY_OFFSET(src));

		- split = set_blocks(btree_bset_first(n1),
                        block_bytes(n1->c)) > (btree_blocks(b) * 4) / 5;

	- btree insert
	- key_buffer

### writeback

- 子主题 1

### req

- insert
- lookup
- cached_dev

	- cached_dev_submit_bio

		- 1172         if (unlikely((d->c && test_bit(CACHE_SET_IO_DISABLE, &d->c->flags)) ||
1173                      dc->io_disable)) {
1174                 bio->bi_status = BLK_STS_IOERR;
1175                 bio_endio(bio);
1176                 return BLK_QC_T_NONE;
1177         }
		- 1179         if (likely(d->c)) {
1180                 if (atomic_read(&d->c->idle_counter))
1181                         atomic_set(&d->c->idle_counter, 0);
1182                 /*
1183                  * If at_max_writeback_rate of cache set is true and new I/O
1184                  * comes, quit max writeback rate of all cached devices
1185                  * attached to this cache set, and set at_max_writeback_rate
1186                  * to false.
1187                  */
1188                 if (unlikely(atomic_read(&d->c->at_max_writeback_rate) == 1)) {
1189                         atomic_set(&d->c->at_max_writeback_rate, 0);
1190                         quit_max_writeback_rate(d->c, dc);
1191                 }
1192         }

			- 设置idle_counter到0，idle_counter是用来判断是否可以设置writeback 到max的。逻辑如下：

 132         /*
 133          * Idle_counter is increased everytime when update_writeback_rate() is
 134          * called. If all backing devices attached to the same cache set have
 135          * identical dc->writeback_rate_update_seconds values, it is about 6
 136          * rounds of update_writeback_rate() on each backing device before
 137          * c->at_max_writeback_rate is set to 1, and then max wrteback rate set
 138          * to each dc->writeback_rate.rate.
 139          * In order to avoid extra locking cost for counting exact dirty cached
 140          * devices number, c->attached_dev_nr is used to calculate the idle
 141          * throushold. It might be bigger if not all cached device are in write-
 142          * back mode, but it still works well with limited extra rounds of
 143          * update_writeback_rate().
 144          */
 145         if (atomic_inc_return(&c->idle_counter) <
 146             atomic_read(&c->attached_dev_nr) * 6)
 147                 return false;
			- quite_max_writeback_rate() 主要是设置所有backing的writeback rate到最小值，也就是1. 

		- 1194         bio_set_dev(bio, dc->bdev);
1195         bio->bi_iter.bi_sector += dc->sb.data_offset;

设置bio的dev为backing device，并且bio的offset 往后移动一个data_offset，这个data_offset是用来存储bcache的superblock的。（data_offset: 8K superblock [4K-8K]）
		- if (cached_dev_get(dc))

			- 如果获得了cached_dev的reference，
s = search_alloc(bio, d);

				- 715 static inline struct search *search_alloc(struct bio *bio,
 716                                           struct bcache_device *d)
 717 {
 718         struct search *s;
 719 
 720         s = mempool_alloc(&d->c->search, GFP_NOIO);
 721 
 722         closure_init(&s->cl, NULL);
 723         do_bio_hook(s, bio, request_endio);
 724         atomic_inc(&d->c->search_inflight);
 725 
 726         s->orig_bio             = bio;
 727         s->cache_miss           = NULL;
 728         s->cache_missed         = 0;
 729         s->d                    = d;
 730         s->recoverable          = 1;
 731         s->write                = op_is_write(bio_op(bio));
 732         s->read_dirty_data      = 0;
 733         /* Count on the bcache device */
 734         s->start_time           = disk_start_io_acct(d->disk, bio_sectors(bio), bio_op(bio));
 735         s->iop.c                = d->c;
 736         s->iop.bio              = NULL;
 737         s->iop.inode            = d->id;
 738         s->iop.write_point      = hash_long((unsigned long) current, 16);
 739         s->iop.write_prio       = 0;
 740         s->iop.status           = 0;
 741         s->iop.flags            = 0;
 742         s->iop.flush_journal    = op_is_flush(bio->bi_opf);
 743         s->iop.wq               = bcache_wq;
 744 
 745         return s;
 746 }

do_bio_hook(s, bio, request_endio) clone bio，并且设置endio

					-  682 static void do_bio_hook(struct search *s,
     /* [previous][next][first][last][top][bottom][index][help]  */
 683                         struct bio *orig_bio,
 684                         bio_end_io_t *end_io_fn)
 685 {
 686         struct bio *bio = &s->bio.bio;
 687 
 688         bio_init(bio, NULL, 0);
 689         __bio_clone_fast(bio, orig_bio);
 690         /*
 691          * bi_end_io can be set separately somewhere else, e.g. the
 692          * variants in,
 693          * - cache_bio->bi_end_io from cached_dev_cache_miss()
 694          * - n->bi_end_io from cache_lookup_fn()
 695          */
 696         bio->bi_end_io          = end_io_fn;
 697         bio->bi_private         = &s->cl;
 698 
 699         bio_cnt_set(bio, 3);
 700 }

search在mempool_alloc的时候会alloc 一个bbio，里面包好一个bio。
（1）bio_init这个bio。
（2）把原本的bio clone到bio->bio里面。这样原本上层发下来的bio就有了两个，一个是s->bio->bio，另一个后面会设置到s->orig_bio里面。

			- 1201                 if (!bio->bi_iter.bi_size) {
1202                         /*
1203                          * can't call bch_journal_meta from under
1204                          * submit_bio_noacct
1205                          */
1206                         continue_at_nobarrier(&s->cl,
1207                                               cached_dev_nodata,
1208                                               bcache_wq);

如果没有数据，就走cached_dev_nodata。

				- cached_dev_nodata

					- 1060         if (s->iop.flush_journal)
1061                 bch_journal_meta(s->iop.c, cl);

如果是flush，需要做一次bch_journal_meta，将journal flush到cache device里面。
					- 1063         /* If it's a flush, we send the flush to the backing device too */
1064         bio->bi_end_io = backing_request_endio;
1065         closure_bio_submit(s->iop.c, bio, cl);

flush 还需要将flush bio 也发送到backing device，实际上没必要？
					- continue_at(cl, cached_dev_bio_complete, NULL);

						-  750 static void cached_dev_bio_complete(struct closure *cl)
 751 {
 752         struct search *s = container_of(cl, struct search, cl);
 753         struct cached_dev *dc = container_of(s->d, struct cached_dev, disk);
 754 
 755         cached_dev_put(dc);
 756         search_free(cl);
 757 }

			- 1209                 } else {
1210                         s->iop.bypass = check_should_bypass(dc, bio);
1211 
1212                         if (rw)
1213                                 cached_dev_write(dc, s);
1214                         else
1215                                 cached_dev_read(dc, s);
1216                 }

				- check_should_bypass

bypass 为什么还要调用cached_dev_write, 而不是调用detached_dev_do_request()， 因为就算是bypass，还需要invalidate相应的cache数据。

					-  371         if (test_bit(BCACHE_DEV_DETACHING, &dc->disk.flags) ||
 372             c->gc_stats.in_use > CUTOFF_CACHE_ADD ||
 373             (bio_op(bio) == REQ_OP_DISCARD))
 374                 goto skip;

（1） 如果在detaching 状态，直接bypass
（2）如果in_use 超过了 95，直接bypass
（3） 如果是discard，也直接bypass
					-  376         if (mode == CACHE_MODE_NONE ||
 377             (mode == CACHE_MODE_WRITEAROUND &&
 378              op_is_write(bio_op(bio))))
 379                 goto skip;

如果cache_mode 是none 或者 cache_mode 是writearound同时op是写，那么直接bypass。
					-  391         if ((bio->bi_opf & (REQ_RAHEAD|REQ_BACKGROUND))) {
 392                 if (!(bio->bi_opf & (REQ_META|REQ_PRIO)) &&
 393                     (dc->cache_readahead_policy != BCH_CACHE_READA_ALL))
 394                         goto skip;
 395         }

如果不是业务请求的读，（预读或者后台读），这个时候需要进行下面的判断：
（1） 如果配置的readahead_policy是 BCH_CACHE_READA_META 那么只有metadata才会从cached_dev_read, 数据的预读会直接bypass。这样可以节省cache空间。
（2）如果readahead_policy 是BCH_CACHE_READA_ALL，所有的数据都走cached_dev_read，会放到cache里面。
					-  397         if (bio->bi_iter.bi_sector & (c->cache->sb.block_size - 1) ||
 398             bio_sectors(bio) & (c->cache->sb.block_size - 1)) {
 399                 pr_debug("skipping unaligned io\n");
 400                 goto skip;
 401         }

如果不是block_size 对其的IO，都直接bypass。
					-  410         congested = bch_get_congested(c);
 411         if (!congested && !dc->sequential_cutoff)
 412                 goto rescale;

如果没有出现拥塞，而且sequential_cutoff 配置的0. 直接做rescale prio然后开始IO。

反过来说，如果发生了拥塞，那就需要下面的统计。
或者sequentail_cutoff 配置了值，也需要下面的统计。
					- congested 和 dc->sequential_cutoff 如果有一个不为零，下面就根据这里面的值来判断是否应该bypass，后续有涉及到之后在补充，暂时都不使用。
					- 如果不需要bypass的IO，不管拥塞或者sequential 与否，都需要执行一个function：bch_rescale_priorities(struct cache_set *c, int sectors) 

						-   90         unsigned long next = c->nbuckets * c->cache->sb.bucket_size / 1024;
  91         int r;
  92 
  93         atomic_sub(sectors, &c->rescale);
  94 
  95         do {
  96                 r = atomic_read(&c->rescale);
  97 
  98                 if (r >= 0)
  99                         return;
 100         } while (atomic_cmpxchg(&c->rescale, r, r + next) != r);

每读写量间隔cache 总容量的1/1024，就需要做一次真正的rescale。
（1） 使用c->rescale 记录当前周期还有多少容量没有消耗
（2）使用rescale 减掉 这次IO的sector，如果rescale仍然大于0，说明还有空间没有被消耗，这个周期还没有使用掉总容量的1/1024，不需要rescale，直接返回。
（3）如果小于0，说明这个周期的容量使用完了，需要给rescale 加上1/1024，这个时候需要考虑其他进程也在做相同的操作，防止race，所以使用了atomic_cmpxchg，这个函数比较c->rescale和 r是否相同，如果相同，吧c->rescale 设置成为r + next。 如果不同则不操作。函数返回值是c->rescale 的旧值，也就是说我们可以通过这个返回值和r进行比较来知道这次操作有没有执行。如果返回不等于r，说明有其他人已经改变了rescale，我们这次设置没有执行，那么进入下一个循环。如果等于r，说明我们这次设置成功，那我们这次需要真正的去rescale。
						-  104         c->min_prio = USHRT_MAX;
 105 
 106         ca = c->cache;
 107         for_each_bucket(b, ca)
 108                 if (b->prio &&
 109                     b->prio != BTREE_PRIO &&
 110                     !atomic_read(&b->pin)) {
 111                         b->prio--;
 112                         c->min_prio = min(c->min_prio, b->prio);
 113                 }
 114 
 115         mutex_unlock(&c->bucket_lock);
真正的rescale 很简单，也比较耗时，主要是把所有bucket的prio 属性都进行减小。

我们会在invalidate的时候使用到prio，lru算法。

				- cached_dev_read

					-  955         struct closure *cl = &s->cl;
 956 
 957         closure_call(&s->iop.cl, cache_lookup, NULL, cl);
 958         continue_at(cl, cached_dev_read_done_bh, NULL);

						- closure_call(&s->iop.cl, cache_lookup, NULL, cl);

							-  585         bch_btree_op_init(&s->op, -1);
 586 
 587         ret = bch_btree_map_keys(&s->op, s->iop.c,
 588                                  &KEY(s->iop.inode, bio->bi_iter.bi_sector, 0),
 589                                  cache_lookup_fn, MAP_END_KEY);

遍历所有的key，然后执行cache_lookup_fn。MAP_END_KEY表示要对于level 等于0的btree执行cache_lookup_fn。

找的key是 &KEY(s->iop.inode, bio->bi_iter.bi_sector, 0)， 也就是需要读的数据开头位置。

								- cache_lookup_fn

									-  520         if (bkey_cmp(k, &KEY(s->iop.inode, bio->bi_iter.bi_sector, 0)) <= 0)
 521                 return MAP_CONTINUE;

如果发现当前btree 的key 和我们的期望值相比较，如果btree的key小于我们想要找的key，说明这个btree 的数据区域结尾小于我们想要找的区域开头，直接跳过这个btree。
									-  523         if (KEY_INODE(k) != s->iop.inode ||
 524             KEY_START(k) > bio->bi_iter.bi_sector) {
 525                 unsigned int bio_sectors = bio_sectors(bio);
 526                 unsigned int sectors = KEY_INODE(k) == s->iop.inode
 527                         ? min_t(uint64_t, INT_MAX,
 528                                 KEY_START(k) - bio->bi_iter.bi_sector)
 529                         : INT_MAX;
 530                 int ret = s->d->cache_miss(b, s, bio, sectors);
 531 
 532                 if (ret != MAP_CONTINUE)
 533                         return ret;
 534 
 535                 /* if this was a complete miss we shouldn't get here */
 536                 BUG_ON(bio_sectors <= sectors);
 537         }

两种情况我们认为现在出现了miss：
（1） 当前btree 的inode 和我们期望的inode 不一致，这个时候我们认为miss。
在map_key的时候，会先去search一下key，也就是说，我们会先跳过整个全局btree前面inode不相同的bkey，我们第一个开始执行 cache_lookup_fn的bkey就是inode相同的bkey。

如果inode不相等，有两种情况：
1.1 整个btree里面没有这个inode，那么第一个bkey就相同，那么我们确实miss。
1.2 开始有bkey是相同的inode，然后这个inode的bkey结束了，进入下一个inode的bkey，也就是相同inode的cache 数据，已经对比结束了，那也是miss。



										- cached_dev_cache_miss

											-  880         int ret = MAP_CONTINUE;
 881         unsigned int reada = 0;
 882         struct cached_dev *dc = container_of(s->d, struct cached_dev, disk);
 883         struct bio *miss, *cache_bio;
 884 
 885         s->cache_missed = 1;

首先标记cache_missed等于1，这个flag会在后面统计miss信息的。
											-  887         if (s->cache_miss || s->iop.bypass) {
 888                 miss = bio_next_split(bio, sectors, GFP_NOIO, &s->d->bio_split);
 889                 ret = miss == bio ? MAP_DONE : MAP_CONTINUE;
 890                 goto out_submit;
 891         }

如果cache_miss不为空，说明已经有一个cache_miss在执行，那么直接submit就可以了。
											- 子主题 3

									-  539         if (!KEY_SIZE(k))
 540                 return MAP_CONTINUE;
									-  542         /* XXX: figure out best pointer - for multiple cache devices */
 543         ptr = 0;
 544 
 545         PTR_BUCKET(b->c, k, ptr)->prio = INITIAL_PRIO;
 546 
 547         if (KEY_DIRTY(k))
 548                 s->read_dirty_data = true;
 549 
 550         n = bio_next_split(bio, min_t(uint64_t, INT_MAX,
 551                                       KEY_OFFSET(k) - bio->bi_iter.bi_sector),
 552                            GFP_NOIO, &s->d->bio_split);

新建一个bio，从bio_split这个bioset里面创建一个bio，这个bio带有一个front_pad，就是bbio。也就是说，bio_split这个bioset里面的每一个bio都被一个bbio包裹着。

1909         if (bioset_init(&c->bio_split, 4, offsetof(struct bbio, bio),
1910                         BIOSET_NEED_BVECS|BIOSET_NEED_RESCUER))

这个是bioset init的时候，第三个参数，offsetof(struct bbio, bio)， 就是一个front_pad，也就是说每一个从这个bioset里面创建的bio，都带有一段前缀，大小为offsetof(struct bbio, bio)。 所以bbio里面bio一定是最后一个member。

这里还要提一下，bio_next_split返回的bio，里面使用的vector对应的还是以前的空间，没有重新分配内存，也就是说，如果我们将bio split成为若干个bio之后，读出来的数据不需要重新整合，已经整合到原始bio期望的内存空间了。
									-  554         bio_key = &container_of(n, struct bbio, bio)->key;
 555         bch_bkey_copy_single_ptr(bio_key, k, ptr);
 556 
 557         bch_cut_front(&KEY(s->iop.inode, n->bi_iter.bi_sector, 0), bio_key);
 558         bch_cut_back(&KEY(s->iop.inode, bio_end_sector(n), 0), bio_key);
 559 
 560         n->bi_end_io    = bch_cache_read_endio;
 561         n->bi_private   = &s->cl;

现在需要初始化这个bbio的key
（1） 设置读取位置： bch_bkey_copy_single_ptr(bio_key, k, ptr)
（2） 设置读取大小，把前后都给cut掉。
（3）设置endio和private，会在endio里面使用。
									-  574         __bch_submit_bbio(n, b->c);
 575         return n == bio ? MAP_DONE : MAP_CONTINUE;

提交这个bbio到cache盘，如果我们发现n 和bio 是一个，就说明我们在bio_next_split的时候没有更多的空间留下来了，整个bio都返回给n了。那就直接返回MAP_DONE。否则，说明还有需要读取的地方，返回continue

						- continue_at(cl, cached_dev_read_done_bh, NULL);

							-  869         if (s->iop.status)
 870                 continue_at_nobarrier(cl, cached_dev_read_error, bcache_wq);

如果发现IO 出现error，就调用cached_dev_read_error处理。
							-  871         else if (s->iop.bio || verify(dc))
 872                 continue_at_nobarrier(cl, cached_dev_read_done, bcache_wq);

如果是cache miss或者需要verify，就调用cached_dev_read_done处理
							-  873         else
 874                 continue_at_nobarrier(cl, cached_dev_bio_complete, NULL);

如果正常从cache里面独处数据，直接调用cached_dev_bio_complete。也就是说，如果我们需要对checksum 进行校验，需要在这个流程里面，当然，需要读取的数据是整个key的区域。

				- cached_dev_write

					- bch_keybuf_check_overlapping(&s->iop.c->moving_gc_keys, &start, &end);

检查当前写入区域是否正在做moving_gc，如果是，将这段key从moving_gc中删除。也就是说正在被修改的内容，不能做movinggc。
					-  981         down_read_non_owner(&dc->writeback_lock);

拿到writeback_lock，主要是对于writeback的keys需要做一些查询和修改。
 982         if (bch_keybuf_check_overlapping(&dc->writeback_keys, &start, &end)) {
 983                 /*
 984                  * We overlap with some dirty data undergoing background
 985                  * writeback, force this write to writeback
 986                  */
 987                 s->iop.bypass = false;
 988                 s->iop.writeback = true;
 989         }

如果需要修改的区域正在执行writeback，那么这部分数据从writeback_keys里面删除。
并且，设置这次写入为writeback。也就是说这次IO必须要写入到cache中，修改了cache之后，writeback回到backing。即使现在是writearound模式，如果我们修改正在writeback的数据，也会先进入cachedevice，然后writeback到backing里面。
					-  991         /*
 992          * Discards aren't _required_ to do anything, so skipping if
 993          * check_overlapping returned true is ok
 994          *
 995          * But check_overlapping drops dirty keys for which io hasn't started,
 996          * so we still want to call it.
 997          */
 998         if (bio_op(bio) == REQ_OP_DISCARD)
 999                 s->iop.bypass = true;

上面设置了bypass为false，但是有一个例外，那就是discard。就算discard的数据正在做writeback，也需要直接bypass给backing，然后把cache 的datainvalidate掉就可以了。
					- 1001         if (should_writeback(dc, s->orig_bio,
1002                              cache_mode(dc),
1003                              s->iop.bypass)) {
1004                 s->iop.bypass = false;
1005                 s->iop.writeback = true;
1006         }

这里有一点confusing，为什么有两个函数来设置bypass和writeback？
check_should_bypass是针对IO本身，但是should_writeback针对当前cache状态。

应该是以bypass为主，should_writeback只是一个辅助。

两个不是互斥的，比如discard正在writeback的区域，那么会得到bypss=true，writeback=true.但是接下来处理的时候可以看到，会优先考虑bypass。
					- 1008         if (s->iop.bypass) {
1009                 s->iop.bio = s->orig_bio;
1010                 bio_get(s->iop.bio);
1011 
1012                 if (bio_op(bio) == REQ_OP_DISCARD &&
1013                     !blk_queue_discard(bdev_get_queue(dc->bdev)))
1014                         goto insert_data;
1015 
1016                 /* I/O request sent to backing device */
1017                 bio->bi_end_io = backing_request_endio;
1018                 closure_bio_submit(s->iop.c, bio, cl);
1019 
1020         }

如果需要bypass，首先设置iop的bio为orig_bio。这是用户发下来的bio。
在发送之前bio_get一下这个用户发下来的bio，等到io结束的时候put一下，在backing_request_endio里面。

如果是DISCARD，直接向backing设备发送DISCARD命令，然后goto到insert_data，目的是为了invalidate cache的data。

end_io设置成backing_reqeust_endio

注意，closure_bio_submit()提交的bio是clone出来的bio。
					- if (s->iop.writeback) {
1021                 bch_writeback_add(dc);
1022                 s->iop.bio = bio;
1023 
1024                 if (bio->bi_opf & REQ_PREFLUSH) {
1025                         /*
1026                          * Also need to send a flush to the backing
1027                          * device.
1028                          */
1029                         struct bio *flush;
1030 
1031                         flush = bio_alloc_bioset(GFP_NOIO, 0,
1032                                                  &dc->disk.bio_split);
1033                         if (!flush) {
1034                                 s->iop.status = BLK_STS_RESOURCE;
1035                                 goto insert_data;
1036                         }
1037                         bio_copy_dev(flush, bio);
1038                         flush->bi_end_io = backing_request_endio;
1039                         flush->bi_private = cl;
1040                         flush->bi_opf = REQ_OP_WRITE | REQ_PREFLUSH;
1041                         /* I/O request sent to backing device */
1042                         closure_bio_submit(s->iop.c, flush, cl);
1043                 }
1044         } 

如果不是bypass，设置了writeback，这个时候需要这些操作。
统计writeback量，如果是FLUSH，需要向backing发送一个flush的bio。TODO这个地方不确定原因是什么。

设置iop.bio是clone出来的bio

						-  130 static inline void bch_writeback_add(struct cached_dev *dc)
     /* [previous][next][first][last][top][bottom][index][help]  */
 131 {
 132         if (!atomic_read(&dc->has_dirty) &&
 133             !atomic_xchg(&dc->has_dirty, 1)) {
 134                 if (BDEV_STATE(&dc->sb) != BDEV_STATE_DIRTY) {
 135                         SET_BDEV_STATE(&dc->sb, BDEV_STATE_DIRTY);
 136                         /* XXX: should do this synchronously */
 137                         bch_write_bdev_super(dc, NULL);
 138                 }
 139 
 140                 bch_writeback_queue(dc);
 141         }
 142 }
增加dirty，然后wakeup writebackthread

					- else {
1045                 s->iop.bio = bio_clone_fast(bio, GFP_NOIO, &dc->disk.bio_split);
1046                 /* I/O request sent to backing device */
1047                 bio->bi_end_io = backing_request_endio;
1048                 closure_bio_submit(s->iop.c, bio, cl);

如果bypass和writeback都是false，那么IO会直接发送到backing。TODO 为什么需要在clone？而且什么场景会出现bypass和writeback都是false？
					- closure_call(&s->iop.cl, bch_data_insert, NULL, cl); 
开始插入数据到data，不管是bypass还是writeback或者其他的情况，都需要走到这个地方。

						-  308 void bch_data_insert(struct closure *cl)
     /* [previous][next][first][last][top][bottom][index][help]  */
 309 {
 310         struct data_insert_op *op = container_of(cl, struct data_insert_op, cl);
 311 
 312         trace_bcache_write(op->c, op->inode, op->bio,
 313                            op->writeback, op->bypass);
 314 
 315         bch_keylist_init(&op->insert_keys);
 316         bio_get(op->bio);
 317         bch_data_insert_start(cl);
 318 }

可以看到这个地方有一个trace点，也就是说所有的bcachewrite 都会走到这个地方。

（1）新建一个insert_keys的keylist，这个是用来存储所有的insert keys，有可能一个write里面需要insert多个key
（2）bio_get在最后endio的时候回去put
（3）实际insert是调用bch_data_insert_start()

							-  189         struct data_insert_op *op = container_of(cl, struct data_insert_op, cl);
 190         struct bio *bio = op->bio, *n;
 191 
 192         if (op->bypass)
 193                 return bch_data_invalidate(cl);

首先拿到一个data_insert_op，也就是说现在在做datainsert，我们不再需要search这么大的数据结构，只需要s->iop就可以了，那么iop里面的bio就是实际要插入的数据的bio。

如果是bypass，而且走到了data_insert，那么说明我们需要invalidate这部分数据，比如detaching的时候。

								-  108 static void bch_data_invalidate(struct closure *cl)
     /* [previous][next][first][last][top][bottom][index][help]  */
 109 {
 110         struct data_insert_op *op = container_of(cl, struct data_insert_op, cl);
 111         struct bio *bio = op->bio;
 112 
 113         pr_debug("invalidating %i sectors from %llu\n",
 114                  bio_sectors(bio), (uint64_t) bio->bi_iter.bi_sector);
 115 
 116         while (bio_sectors(bio)) {
 117                 unsigned int sectors = min(bio_sectors(bio),
 118                                        1U << (KEY_SIZE_BITS - 1));
 119 
 120                 if (bch_keylist_realloc(&op->insert_keys, 2, op->c))
 121                         goto out;
 122 
 123                 bio->bi_iter.bi_sector  += sectors;
 124                 bio->bi_iter.bi_size    -= sectors << 9;
 125 
 126                 bch_keylist_add(&op->insert_keys,
 127                                 &KEY(op->inode,
 128                                      bio->bi_iter.bi_sector,
 129                                      sectors));
 130         }
 131 
 132         op->insert_data_done = true;
 133         /* get in bch_data_insert() */
 134         bio_put(bio);
 135 out:
 136         continue_at(cl, bch_data_insert_keys, op->wq);

每个key的最大size是65535个sectore，所以每次invalidate的大小也就是这么多。

							-  195         if (atomic_sub_return(bio_sectors(bio), &op->c->sectors_to_gc) < 0)
 196                 wake_up_gc(op->c);

减小sectore，如果需要wake_up_gc，这里涉及到一个wakeup gc的逻辑，bcache里面写入1/16的数据，就回去wakeup一下gc。这样可以间隔的去启动gc。
							-  198         /*
 199          * Journal writes are marked REQ_PREFLUSH; if the original write was a
 200          * flush, it'll wait on the journal write.
 201          */
 202         bio->bi_opf &= ~(REQ_PREFLUSH|REQ_FUA);

如果bio需要flush，那么这个地方过滤掉，因为如果是flush的bio，在search_alloc里面会设置。
s->iop.flush_journal    = op_is_flush(bio->bi_opf);

然后我们会在data insert结束之后，insertkey的时候flush journal，这样来保证flush。这里有一个前提，就是journal和data使用的是同一个设备，否则journal设备的flush不会作用于data。
							- 204         do {
 205                 unsigned int i;
 206                 struct bkey *k;
 207                 struct bio_set *split = &op->c->bio_split;
 208 
 209                 /* 1 for the device pointer and 1 for the chksum */
 210                 if (bch_keylist_realloc(&op->insert_keys,
 211                                         3 + (op->csum ? 1 : 0),
 212                                         op->c)) {
 213                         continue_at(cl, bch_data_insert_keys, op->wq);
 214                         return;
 215                 }

首先在insert_keys里面添加一个key，大小为3+（op->csum? 1:0）。 
（1）如果csum关闭，key的大小为3个u64。包括一个high，一个low。再加上一个ptr。bcache支持一个cachedevice的cache_set。
（2）如果csum打开，在ptr的第二个u64存储checksum。
							- 216 
 217                 k = op->insert_keys.top;

top指向realloc之前的尾部，也就是说，现在只想的就是我们新加入的key。
 218                 bkey_init(k);
 219                 SET_KEY_INODE(k, op->inode);
 220                 SET_KEY_OFFSET(k, bio->bi_iter.bi_sector);

设置inode和offset，注意，这里设置的offset是bi_sector，但是实际bkey的offset是bi_sector+size。这个会再随后更新。
							-  221 
 222                 if (!bch_alloc_sectors(op->c, k, bio_sectors(bio),
 223                                        op->write_point, op->write_prio,
 224                                        op->writeback))
 225                         goto err;
 226 

分配空间给这个bkey，这个空间会在data_bucket里面分配。

								-  597 /*
 598  * Allocates some space in the cache to write to, and k to point to the newly
 599  * allocated space, and updates KEY_SIZE(k) and KEY_OFFSET(k) (to point to the
 600  * end of the newly allocated space).
 601  *
 602  * May allocate fewer sectors than @sectors, KEY_SIZE(k) indicates how many
 603  * sectors were actually allocated.
 604  *
 605  * If s->writeback is true, will not fail.
 606  */
 607 bool bch_alloc_sectors(struct cache_set *c,
     /* [previous][next][first][last][top][bottom][index][help]  */
 608                        struct bkey *k,
 609                        unsigned int sectors,
 610                        unsigned int write_point,
 611                        unsigned int write_prio,
 612                        bool wait)
 613 {
 614         struct open_bucket *b;
 615         BKEY_PADDED(key) alloc;
 616         unsigned int i;
								-  618         /*
 619          * We might have to allocate a new bucket, which we can't do with a
 620          * spinlock held. So if we have to allocate, we drop the lock, allocate
 621          * and then retry. KEY_PTRS() indicates whether alloc points to
 622          * allocated bucket(s).
 623          */
 624 
 625         bkey_init(&alloc.key);
 626         spin_lock(&c->data_bucket_lock);

								-  628         while (!(b = pick_data_bucket(c, k, write_point, &alloc.key))) {
 629                 unsigned int watermark = write_prio
 630                         ? RESERVE_MOVINGGC
 631                         : RESERVE_NONE;
 632 
 633                 spin_unlock(&c->data_bucket_lock);
 634 
 635                 if (bch_bucket_alloc_set(c, watermark, &alloc.key, wait))
 636                         return false;
 637 
 638                 spin_lock(&c->data_bucket_lock);
 639         }

（1）先从open_data_bucket里面选择一个满足条件的bucket，如果选到，推出循环。
（2）如果没有选到，说明需要重新分配bucket，先释放data_bucket_lock。
（3）调用bch_bucket_alloc_set() 分配bucket。
（4）然后重新获得data_bucket_lock，重新进入pick_data_bucket。

									- pick_data_bucket

										-  566 static struct open_bucket *pick_data_bucket(struct cache_set *c,
     /* [previous][next][first][last][top][bottom][index][help]  */
 567                                             const struct bkey *search,
 568                                             unsigned int write_point,
 569                                             struct bkey *alloc)
 570 {
 571         struct open_bucket *ret, *ret_task = NULL;
 572 
 573         list_for_each_entry_reverse(ret, &c->data_buckets, list)
 574                 if (UUID_FLASH_ONLY(&c->uuids[KEY_INODE(&ret->key)]) !=
 575                     UUID_FLASH_ONLY(&c->uuids[KEY_INODE(search)]))
 576                         continue;
 577                 else if (!bkey_cmp(&ret->key, search))
 578                         goto found;
 579                 else if (ret->last_write_point == write_point)
 580                         ret_task = ret;

可以看到，pick data_bucket的时候，从data_buckets_list找到符合要求的data_bucket。规则如下：
（1）从后往前找，为什么要从后往前找，为了更快的命中，每次我们找到合适的bucket之后，都会讲bucket放到list的最后。下一次找的时候，由于局部性原理，更大可能会更快的找到。
（2）如果是flashdev，必须要招flashdev的data_bucket。
（3）如果写入的offset和data_bucket的末尾可以连接上，直接返回。这说明是顺序写，放到一起可以合并。这是第一优先级的。
（3）如果不满足（2），查看data_bucket的write_point，其实就是写入的进程hash，和当前hash对比，如果是同一个进程的写入，优先放到一起。这里有一个优化，内部已经试用，但是社区还没有merge，只选择第一个writepoint匹配的data_bucket。
										-  582         ret = ret_task ?: list_first_entry(&c->data_buckets,
 583                                            struct open_bucket, list);

如果上面for循环没有找到合适的data_bucket，说明这是一个全新的进程在写入一个不连续的空间。这个时候直接选择list的第一个data_bucket，同样，根据局部性原理，这个data_bucket是最近最不常使用的bucket，给新的进程使用最合适。
										-  584 found:
 585         if (!ret->sectors_free && KEY_PTRS(alloc)) {
 586                 ret->sectors_free = c->cache->sb.bucket_size;
 587                 bkey_copy(&ret->key, alloc);
 588                 bkey_init(alloc);
 589         }
判断是否需要使用新分配的bucket，这个主要在第一次没有选择到bucket，然后走到alloc bucket之后，下一次循环进入会使用。
（1） 如果当前bucket free为0，说明bucket已经使用结束。并且有alloc传进来，说明在调用pick_data_bucket之前已经为我们分配好了一个bucket。
（2）这个时候只需要将list里面对应位置的bucket元数据替换成新分配的bucket元数据。
										-  591         if (!ret->sectors_free)
 592                 ret = NULL;
 593 
 594         return ret;

如果选出来的bucket没有free空间，返回NULL，告诉调用者，需要新分配bucket。

									- bch_bucket_alloc_set

										-  524         int ret;
 525 
 526         mutex_lock(&c->bucket_lock);
 527         ret = __bch_bucket_alloc_set(c, reserve, k, wait);
 528         mutex_unlock(&c->bucket_lock);
 529         return ret;

拿到bucket_lock，然后调用__bch_bucket_alloc_set 分配一个bucket

											- __bch_bucket_alloc_set

												-  501         bkey_init(k);
 502 
 503         ca = c->cache;
 504         b = bch_bucket_alloc(ca, reserve, wait);
 505         if (b == -1)
 506                 goto err;
在cache上面分配一个新的bucket，如果wait为1，需要等待bucket知道分配成功。

													-  403         /* fastpath */
 404         if (fifo_pop(&ca->free[RESERVE_NONE], r) ||
 405             fifo_pop(&ca->free[reserve], r))
 406                 goto out;

首先检查freelist里面有没有可用的bucket，如果有，直接返回。

但是有可能会出现freelist为空的情况，这个时候我们需要补充freelist
													-  408         if (!wait) {
 409                 trace_bcache_alloc_fail(ca, reserve);
 410                 return -1;
 411         }
当freelist 是空，如果参数里面wait 是0，说明不需要wait，直接返回错误。
													-  413         do {
 414                 prepare_to_wait(&ca->set->bucket_wait, &w,
 415                                 TASK_UNINTERRUPTIBLE);
 416 
 417                 mutex_unlock(&ca->set->bucket_lock);
 418                 schedule();
 419                 mutex_lock(&ca->set->bucket_lock);
 420         } while (!fifo_pop(&ca->free[RESERVE_NONE], r) &&
 421                  !fifo_pop(&ca->free[reserve], r));
423         finish_wait(&ca->set->bucket_wait, &w);

如果需要wait，那么schedule出去，然后等待freelist里面有可以使用的bucket。

注意，在进入wait之前并没有wakeup allocator这个thread，原因是因为每次从freelist里面拿出一个bucket之后就会去wakeup allocator，而且allocator 会一直把free填满之后才会调度出去，也就是说，我们现在发现freelist已经空了，那么allocator肯定已经运作起来了。
													-   424 out:
 425         if (ca->alloc_thread)
 426                 wake_up_process(ca->alloc_thread);
分配到bucket之后，freelist一定不再满，wakeup alloc_thread去继续填满freelist

														- bch_allocator_thread

															-  324                 /*
 325                  * First, we pull buckets off of the unused and free_inc lists,
 326                  * possibly issue discards to them, then we add the bucket to
 327                  * the free list:
 328                  */
 329                 while (1) {
 330                         long bucket;
 331 
 332                         if (!fifo_pop(&ca->free_inc, bucket))
 333                                 break;
 334 
 335                         if (ca->discard) {
 336                                 mutex_unlock(&ca->set->bucket_lock);
 337                                 blkdev_issue_discard(ca->bdev,
 338                                         bucket_to_sector(ca->set, bucket),
 339                                         ca->sb.bucket_size, GFP_KERNEL, 0);
 340                                 mutex_lock(&ca->set->bucket_lock);
 341                         }
 342 
 343                         allocator_wait(ca, bch_allocator_push(ca, bucket));
 344                         wake_up(&ca->set->btree_cache_wait);
 345                         wake_up(&ca->set->bucket_wait);
 346                 }

alloc_thread 首先会尝试不断的从free_inc 里面pop出bucket，然后push到freelist里面。

（1）如果有discard参数，在push到freelist之前会做一次discard。
（2）push到freelist的时候，如果freelist满了，会进入wait。
（3）如果push成功，需要去wakeup 对freelist 的waiter。主要是两个，一个是data的bucket一个是btree的bucket。
															-  348                 /*
 349                  * We've run out of free buckets, we need to find some buckets
 350                  * we can invalidate. First, invalidate them in memory and add
 351                  * them to the free_inc list:
 352                  */
 353 
 354 retry_invalidate:
 355                 allocator_wait(ca, ca->set->gc_mark_valid &&
 356                                !ca->invalidate_needs_gc);
 357                 invalidate_buckets(ca);

如果现在free_inc 已经空了，那么需要去invalidate bucket 来补充free_inc，这个过程有两个要求：
（1）gc_mark_valid，gc执行过程中，我们不能去invalidate。其实这个限制有点问题，原意是担心metadata在gc过程中会被修改，所以不能使用metadata来做invalidate的依据。但是其实gc只会把不能invalidate的那些bucket重新扫描一遍，变成可以invalidate的。也就是所在gc之前就可以invalidate的bucket，gc过程中以及gc之后还是可以invalidate的。TODO，我的patch就是这个思路，可以解决分配等待的问题，导致IOhang
（2）如果invalidate_needs_gc设置了，我们也需要等待，意思是说已经尝试过一次invalidate，发现需要gc，现在gc没有结束，先等着把。gc结束之后，invalidate_needs_gc会cleat掉。

																- invalidate_buckets

																	-  267         BUG_ON(ca->invalidate_needs_gc);
 268 
 269         switch (CACHE_REPLACEMENT(&ca->sb)) {
 270         case CACHE_REPLACEMENT_LRU:
 271                 invalidate_buckets_lru(ca);
 272                 break;
 273         case CACHE_REPLACEMENT_FIFO:
 274                 invalidate_buckets_fifo(ca);
 275                 break;
 276         case CACHE_REPLACEMENT_RANDOM:
 277                 invalidate_buckets_random(ca);
 278                 break;
 279         }

																		- invalidate_buckets_lru

																			-  181         struct bucket *b;
 182         ssize_t i;
 183 
 184         ca->heap.used = 0;
 185 
 186         for_each_bucket(b, ca) {
 187                 if (!bch_can_invalidate_bucket(ca, b))
 188                         continue;
 189 
 190                 if (!heap_full(&ca->heap))
 191                         heap_add(&ca->heap, b, bucket_max_cmp);
 192                 else if (bucket_max_cmp(b, heap_peek(&ca->heap))) {
 193                         ca->heap.data[0] = b;
 194                         heap_sift(&ca->heap, 0, bucket_max_cmp);
 195                 }
 196         }

遍历所有的buckets，然后根据META的信息判断是否可以被invalidate，如果可以，加入到heap里面去。
（1）加入到heap是通过bucket的prio 来排序的，heap里面的data[0] 是heap里面prio 最大的一个bucket。prio 越大表示说明越近访问过这个bucket。
（2）如果heap满了，比较一下当前bucket和peak上面的bucket，如果prio 比peak 小，说明当前的bucket更远访问，那么久更需要被invalidate，替换当前的peek，然后做一次sift。这样保证最大的prio的bucket还在peek。
																			-  198         for (i = ca->heap.used / 2 - 1; i >= 0; --i)
 199                 heap_sift(&ca->heap, i, bucket_min_cmp);

通过sift，将heap里面prio 最小的bucket放到peek里面。下面去invalidate的时候，从prio 最小的开始。
																			-  201         while (!fifo_full(&ca->free_inc)) {
 202                 if (!heap_pop(&ca->heap, b, bucket_min_cmp)) {
 203                         /*
 204                          * We don't want to be calling invalidate_buckets()
 205                          * multiple times when it can't do anything
 206                          */
 207                         ca->invalidate_needs_gc = 1;
 208                         wake_up_gc(ca->set);
 209                         return;
 210                 }
 211 
 212                 bch_invalidate_one_bucket(ca, b);
 213         }
不断的从heap里面pop出来prio最小的一个bucket，然后做invalidate，invalide之后，会将这个bucket加入到free_inc里面。

如果heap里面的bucket都invalidate完了，但是free_inc还没有满，说明现在可以invalidate的bucket不够，需要wakeup一下gc。

																		- invalidate_buckets_fifo

																			-  221         while (!fifo_full(&ca->free_inc)) {
 222                 if (ca->fifo_last_bucket <  ca->sb.first_bucket ||
 223                     ca->fifo_last_bucket >= ca->sb.nbuckets)
 224                         ca->fifo_last_bucket = ca->sb.first_bucket;
 225 
 226                 b = ca->buckets + ca->fifo_last_bucket++;
 227 
 228                 if (bch_can_invalidate_bucket(ca, b))
 229                         bch_invalidate_one_bucket(ca, b);
 230 
 231                 if (++checked >= ca->sb.nbuckets) {
 232                         ca->invalidate_needs_gc = 1;
 233                         wake_up_gc(ca->set);
 234                         return;
 235                 }
 236         }

这个逻辑比较简单，每次从last_bucket开始，往后invalide，如果轮询过一遍都没有满足要求，需要wakeup gc。

																		- invalidate_buckets_random

																			-  241         struct bucket *b;
 242         size_t checked = 0;
 243 
 244         while (!fifo_full(&ca->free_inc)) {
 245                 size_t n;
 246 
 247                 get_random_bytes(&n, sizeof(n));
 248 
 249                 n %= (size_t) (ca->sb.nbuckets - ca->sb.first_bucket);
 250                 n += ca->sb.first_bucket;
 251 
 252                 b = ca->buckets + n;
 253 
 254                 if (bch_can_invalidate_bucket(ca, b))
 255                         bch_invalidate_one_bucket(ca, b);
 256 
 257                 if (++checked >= ca->sb.nbuckets / 2) {
 258                         ca->invalidate_needs_gc = 1;
 259                         wake_up_gc(ca->set);
 260                         return;
 261                 }
 262         }

这个逻辑更简单，随机找一个bucket，尝试能不能invalidate，然后轮询一遍还不能满足free_inc，wakeupgc

															-  359                 /*
 360                  * Now, we write their new gens to disk so we can start writing
 361                  * new stuff to them:
 362                  */
 363                 allocator_wait(ca, !atomic_read(&ca->set->prio_blocked));
 364                 if (CACHE_SYNC(&ca->sb)) {
 365                         /*
 366                          * This could deadlock if an allocation with a btree
 367                          * node locked ever blocked - having the btree node
 368                          * locked would block garbage collection, but here we're
 369                          * waiting on garbage collection before we invalidate
 370                          * and free anything.
 371                          *
 372                          * But this should be safe since the btree code always
 373                          * uses btree_check_reserve() before allocating now, and
 374                          * if it fails it blocks without btree nodes locked.
 375                          */
 376                         if (!fifo_full(&ca->free_inc))
 377                                 goto retry_invalidate;
 378 
 379                         if (bch_prio_write(ca, false) < 0) {
 380                                 ca->invalidate_needs_gc = 1;
 381                                 wake_up_gc(ca->set);
 382                         }
 383                 }

经过invalidate之后，bucket的gen 修改了，我们需要将这个信息持久化到磁盘上面。

													-  444 
 445         b = ca->buckets + r;
 446 
 447         BUG_ON(atomic_read(&b->pin) != 1);
 448 
 449         SET_GC_SECTORS_USED(b, ca->sb.bucket_size);
 450 
 451         if (reserve <= RESERVE_PRIO) {
 452                 SET_GC_MARK(b, GC_MARK_METADATA);
 453                 SET_GC_MOVE(b, 0);
 454                 b->prio = BTREE_PRIO;
 455         } else {
 456                 SET_GC_MARK(b, GC_MARK_RECLAIMABLE);
 457                 SET_GC_MOVE(b, 0);
 458                 b->prio = INITIAL_PRIO;
 459         }
 460 
 461         if (ca->set->avail_nbuckets > 0) {
 462                 ca->set->avail_nbuckets--;
 463                 bch_update_bucket_in_use(ca->set, &ca->set->gc_stats);
 464         }
 465 
 466         return r;

设置bucket的属性，然后返回给调用者。

												-  508         k->ptr[0] = MAKE_PTR(ca->buckets[b].gen,
 509                              bucket_to_sector(c, b),
 510                              ca->sb.nr_this_dev);
 511 
 512         SET_KEY_PTRS(k, 1);

分配好了bucket之后，需要设置key的ptr，k->ptr[0] 表示在cache上面的位置。
SET_KEY_PTRS 设置key有多少个cache，bcache现在只支持一个cache。

								-  641         /*
 642          * If we had to allocate, we might race and not need to allocate the
 643          * second time we call pick_data_bucket(). If we allocated a bucket but
 644          * didn't use it, drop the refcount bch_bucket_alloc_set() took:
 645          */
 646         if (KEY_PTRS(&alloc.key))
 647                 bkey_put(c, &alloc.key);

如果新allocate了bucket，但是没有使用，由于第二次找到合适的bucket了，那么我们需要在这个地方释放bkey。
								-  649         for (i = 0; i < KEY_PTRS(&b->key); i++)
 650                 EBUG_ON(ptr_stale(c, &b->key, i));

刚分配出来的bucket，或者从data_bucket里面选出来的bucket，不可能是stale的
								-  652         /* Set up the pointer to the space we're allocating: */
 653 
 654         for (i = 0; i < KEY_PTRS(&b->key); i++)
 655                 k->ptr[i] = b->key.ptr[i];
 656 
 657         sectors = min(sectors, b->sectors_free);
 658 
 659         SET_KEY_OFFSET(k, KEY_OFFSET(k) + sectors);
 660         SET_KEY_SIZE(k, sectors);
 661         SET_KEY_PTRS(k, KEY_PTRS(&b->key));
 662 

设置key的各属性。注意 offset 设置的是这段数据区域的末尾。
								-  663         /*
 664          * Move b to the end of the lru, and keep track of what this bucket was
 665          * last used for:
 666          */
 667         list_move_tail(&b->list, &c->data_buckets);
 668         bkey_copy_key(&b->key, k);
 669         b->last_write_point = write_point;
 670 
 671         b->sectors_free -= sectors;
 672 
 673         for (i = 0; i < KEY_PTRS(&b->key); i++) {
 674                 SET_PTR_OFFSET(&b->key, i, PTR_OFFSET(&b->key, i) + sectors);
 675 
 676                 atomic_long_add(sectors,
 677                                 &PTR_CACHE(c, &b->key, i)->sectors_written);
 678         }
 679 
 680         if (b->sectors_free < c->cache->sb.block_size)
 681                 b->sectors_free = 0;
 682 
 683         /*
 684          * k takes refcounts on the buckets it points to until it's inserted
 685          * into the btree, but if we're done with this bucket we just transfer
 686          * get_data_bucket()'s refcount.
 687          */
 688         if (b->sectors_free)
 689                 for (i = 0; i < KEY_PTRS(&b->key); i++)
 690                         atomic_inc(&PTR_BUCKET(c, &b->key, i)->pin);
 691 
 692         spin_unlock(&c->data_bucket_lock);

（1） 把bucket 移动到data_bucket 的尾部，方便下次pick的时候快速找到。
（2）把bucket的key, b->key 更新成当前k，目的是为了下次pick的时候可以判断是否和这个bucket连接。
（3）设置writepoint，目的是为了下次pick的时候可以对比找到相同进程的bucket。
（4） 减少bucket的可用空间。

							-  227                 n = bio_next_split(bio, KEY_SIZE(k), GFP_NOIO, split);
 228 
 229                 n->bi_end_io    = bch_data_insert_endio;
 230                 n->bi_private   = cl;
 231 
 232                 if (op->writeback) {
 233                         SET_KEY_DIRTY(k, true);
 234 
 235                         for (i = 0; i < KEY_PTRS(k); i++)
 236                                 SET_GC_MARK(PTR_BUCKET(op->c, k, i),
 237                                             GC_MARK_DIRTY);
 238                 }
 239 
 240                 SET_KEY_CSUM(k, op->csum);
 241                 if (KEY_CSUM(k))
 242                         bio_csum(n, k);
 243 
 244                 trace_bcache_cache_insert(k);
 245                 bch_keylist_push(&op->insert_keys);
 246 
 247                 bio_set_op_attrs(n, REQ_OP_WRITE, 0);
 248                 bch_submit_bbio(n, op->c, k, 0);
 249         } while (n != bio);

分配好空间之后，不断的将key插入到keylist里面，然后把bio里面的数据写入到key对应的data空间中。

								- bio_csum

									-   42         struct bio_vec bv;
  43         struct bvec_iter iter;
  44         uint64_t csum = 0;
  45 
  46         bio_for_each_segment(bv, bio, iter) {
  47                 void *d = kmap(bv.bv_page) + bv.bv_offset;
  48 
  49                 csum = bch_crc64_update(csum, d, bv.bv_len);
  50                 kunmap(bv.bv_page);
  51         }
  52 
  53         k->ptr[KEY_PTRS(k)] = csum & (~0ULL >> 1);

注意，csum的最前一位是0.

								- bch_submit_bbio

									-   45 void bch_submit_bbio(struct bio *bio, struct cache_set *c,
     /* [previous][next][first][last][top][bottom][index][help]  */
  46                      struct bkey *k, unsigned int ptr)
  47 {
  48         struct bbio *b = container_of(bio, struct bbio, bio);
  49 
  50         bch_bkey_copy_single_ptr(&b->key, k, ptr);
  51         __bch_submit_bbio(bio, c);
  52 }

通过bio拿到bbio，然后copy key的ptr到bbio的key里面。

										-   35 {
  36         struct bbio *b = container_of(bio, struct bbio, bio);
  37 
  38         bio->bi_iter.bi_sector  = PTR_OFFSET(&b->key, 0);
  39         bio_set_dev(bio, PTR_CACHE(c, &b->key, 0)->bdev);
  40 
  41         b->submit_time_us = local_clock_us();
  42         closure_bio_submit(c, bio, bio->bi_private);
  43 }

设置bio的bi_sector为b->key的ptr_offset
然后设置dev为cache device。
然后提交bio

							-  251         op->insert_data_done = true;
 252         continue_at(cl, bch_data_insert_keys, op->wq);
return;

完成了data的insert之后，继续执行key的insert。

								- bch_data_insert_keys

									-   60         struct data_insert_op *op = container_of(cl, struct data_insert_op, cl);
  61         atomic_t *journal_ref = NULL;
  62         struct bkey *replace_key = op->replace ? &op->replace_key : NULL;
  63         int ret;
  64 
  65         if (!op->replace)
  66                 journal_ref = bch_journal(op->c, &op->insert_keys,
  67                                           op->flush_journal ? cl : NULL);

首先判断是否是replace，replace表示从backing里面读出数据，现在需要插入到cache里面。如果不是replace，说明是写入新数据，那么需要加入到journal里面。

flush_journal表示上层发下来的bio是否带有flush flag，如果有flushflag，那么需要flushjournal。

										- bch_journal

									-   69         ret = bch_btree_insert(op->c, &op->insert_keys,
  70                                journal_ref, replace_key);
  71         if (ret == -ESRCH) {
  72                 op->replace_collision = true;
  73         } else if (ret) {
  74                 op->status              = BLK_STS_RESOURCE;
  75                 op->insert_data_done    = true;
  76         }

将bkey插入到btree里面。

										- bch_btree_insert()

											- 2440         struct btree_insert_op op;
2441         int ret = 0;
2442 
2443         BUG_ON(current->bio_list);
2444         BUG_ON(bch_keylist_empty(keys));
2445 
2446         bch_btree_op_init(&op.op, 0);
2447         op.keys         = keys;
2448         op.journal_ref  = journal_ref;
2449         op.replace_key  = replace_key;

首先init 一下btree_insert_op， 重要的有三个元素：
（1）需要insert的keys，这个是实际需要insert的key，当然重要。
（2）journal_ref， 这个会在leaf_dirty的时候用到，目的是告诉journal 是否可以被reclaim。如果key还没有落盘，这部分journal是不可以被reclaim的。
（3）replace_key，这个会在cache_miss的时候插入数据用到
											- 2451         while (!ret && !bch_keylist_empty(keys)) {
2452                 op.op.lock = 0;
2453                 ret = bch_btree_map_leaf_nodes(&op.op, c,
2454                                                &START_KEY(keys->keys),
2455                                                btree_insert_fn);
2456         }

不断将insert_keys里面的key插入到btree里面去，注意，这里使用的是map_leaf，也就是说，只会插入到leaf node。

												- btree_insert_fn

													- 子主题 1

											- 2458         if (ret) {
2459                 struct bkey *k;
2460 
2461                 pr_err("error %i\n", ret);
2462 
2463                 while ((k = bch_keylist_pop(keys)))
2464                         bkey_put(c, k);
2465         } else if (op.op.insert_collision)
2466                 ret = -ESRCH;
2467 
2468         return ret;

如果出现error，需要释放没有完成插入的key。

							-  254 err:
 255         /* bch_alloc_sectors() blocks if s->writeback = true */
 256         BUG_ON(op->writeback);
 257 
 258         /*
 259          * But if it's not a writeback write we'd rather just bail out if
 260          * there aren't any buckets ready to write to - it might take awhile and
 261          * we might be starving btree writes for gc or something.
 262          */
 263 
 264         if (!op->replace) {
 265                 /*
 266                  * Writethrough write: We can't complete the write until we've
 267                  * updated the index. But we don't want to delay the write while
 268                  * we wait for buckets to be freed up, so just invalidate the
 269                  * rest of the write.
 270                  */
 271                 op->bypass = true;
 272                 return bch_data_invalidate(cl);
 273         } else {
 274                 /*
 275                  * From a cache miss, we can just insert the keys for the data
 276                  * we have written or bail out if we didn't do anything.
 277                  */
 278                 op->insert_data_done = true;
 279                 bio_put(bio);
 280 
 281                 if (!bch_keylist_empty(&op->insert_keys))
 282                         continue_at(cl, bch_data_insert_keys, op->wq);
 283                 else
 284                         closure_return(cl);
 285         }

这是一场场景，如果我们分配空间失败，需要怎么处理？
（1）如果是正常写入，一旦cache 空间分配失败，我们直接写入数据到backing。设置bypass为true，并且把cache里面对应的区域invalidate掉。
（2）如果是读出来的数据需要插入，但是没有空间了，我们直接放弃插入，这部分数据是backing里面读出来的，不一定要放到cache中。只是一个读性能优化而已。

		- else 
1220                 /* I/O request sent to backing device */
1221                 detached_dev_do_request(d, bio);

			- 如果 cached_dev_get() 返回0，说明现在的cached_dev 处于detached状态。这个时候IO直接进入backing device就可以了。不需要再走cache 设备。

				- detached_dev_io_private
				- 1111         ddip = kzalloc(sizeof(struct detached_dev_io_private), GFP_NOIO);
1112         ddip->d = d;
1113         /* Count on the bcache device */
1114         ddip->start_time = part_start_io_acct(d->disk, &ddip->part, bio);
1115         ddip->bi_end_io = bio->bi_end_io;
1116         ddip->bi_private = bio->bi_private;
1117         bio->bi_end_io = detached_dev_end_io;
1118         bio->bi_private = ddip;

detached_dev_io_private 做了一个bio的封装，主要是为了统计一下这次IO的时间。
				- 1120         if ((bio_op(bio) == REQ_OP_DISCARD) &&
1121             !blk_queue_discard(bdev_get_queue(dc->bdev)))
1122                 bio->bi_end_io(bio);

如果是discard，并且backing device 不支持discard，直接endio。
				- 1123         else
1124                 submit_bio_noacct(bio);

否则，提交bio

					- bio 结束之后会调用detached_dev_end_io()

						- 1081         struct detached_dev_io_private *ddip;
1082 
1083         ddip = bio->bi_private;
1084         bio->bi_end_io = ddip->bi_end_io;
1085         bio->bi_private = ddip->bi_private;
1086 
1087         /* Count on the bcache device */
1088         part_end_io_acct(ddip->part, bio, ddip->start_time);
1089 
1090         if (bio->bi_status) {
1091                 struct cached_dev *dc = container_of(ddip->d,
1092                                                      struct cached_dev, disk);
1093                 /* should count I/O error for backing device here */
1094                 bch_count_backing_io_errors(dc, bio);
1095         }
1096 
1097         kfree(ddip);
1098         bio->bi_end_io(bio);

(1) 统计这次backing io 的时间记录到bcache disk 统计计数里面。
（2）统计IO error 到backing_io_error里面
（3） 调用bio本来的 end_io。

	- read
	- write

- flash_dev

	- flash_dev_submit_bio

## 生命周期

### bcache_device

- 

### cache_set

- unregister

	- 1839 void bch_cache_set_unregister(struct cache_set *c)
1840 {
1841         set_bit(CACHE_SET_UNREGISTERING, &c->flags);
1842         bch_cache_set_stop(c);
1843 }

		- 当我们显式调用 unregister的时候，表示我们想把这个cache_set拿出来，但是上面使用的bcache_device可以继续运行在writearound模式。所以需要设置UNREGISTERING flag，然后调用cache_set_stop。这个时候就不会去stop cached_dev，只会去detach掉，继续运行在没有cache的模式。

注意，如果是flash_dev，这样的bcache_device是没有backing设备的，如果unregister cache_set，这样的flash_dev只能直接stop。

- stop

	- 1832 void bch_cache_set_stop(struct cache_set *c)
1833 {
1834         if (!test_and_set_bit(CACHE_SET_STOPPING, &c->flags))
1835                 /* closure_fn set to __cache_set_unregister() */
1836                 closure_queue(&c->caching);
1837 }

设置了STOPPING然后queue caching，执行的就是__cache_set_unregister()

		- __cache_set_unregister()

			- 1816                 if (!UUID_FLASH_ONLY(&c->uuids[i]) &&
1817                     test_bit(CACHE_SET_UNREGISTERING, &c->flags)) {
1818                         dc = container_of(d, struct cached_dev, disk);
1819                         bch_cached_dev_detach(dc);
1820                         if (test_bit(CACHE_SET_IO_DISABLE, &c->flags))
1821                                 conditional_stop_bcache_device(c, d, dc);

如果是cached_dev 并且现在是需要去做unregister，那么就去detach，因为你unregister了cache_set 理论上不应该影响backing对应的cacheddev。

既然叫做__cache_set_unregister() ， 为什么还要判断是否有UNREGISTERING flag呢？ 因为cache_set现在只有caching这一个closure 对应停止设备的操作，也就是说 不管是stop还是unregister，都是closure_queue(&c->caching)， 只不过通过一个CACHE_SET_UNREGISTERING来区分对待这两种场景。

如果是flash_dev，只能直接stop，因为flash_dev是纯cache_set构造出来的block_device，没有backing设备，不能运行在没有cache_set的模式下面。

				- bch_cached_dev_detach()

					- 1181          * Block the device from being closed and freed until we're finished
1182          * detaching
1183          */
1184         closure_get(&dc->disk.cl);
1185 
1186         bch_writeback_queue(dc);
1187 
1188         cached_dev_put(dc);

拿了bcache_device的cl，然后启动writeback，当writeback结束的时候，会再去cached_dev_put(dc)。 也就是说，回刷结束之后，我们就会执行detach 结束操作。

						-  899 static inline void cached_dev_put(struct cached_dev *dc)
 900 {
 901         if (refcount_dec_and_test(&dc->count))
 902                 schedule_work(&dc->detach);
 903 }


INIT_WORK(&dc->detach, cached_dev_detach_finish);

只有等到我们writeback 结束了之后才会去schedule_work，注意 refcount_dec_and_test（） 去减小1，并且检查是否为0.如果到0 返回true。

writeback 结束之后执行detach_finish

							- cached_dev_detach_finish

--> bcache_device_detach(&dc->disk);

								- closure_put(&d->c->caching);

这个点很重要，也就是说，bcache_device_detach结束之后才会释放cache_set 的caching，这个时候cache_set_flush才能去执行。

				- conditional_stop_bcache_device

					- if (dc->stop_when_cache_set_failed == BCH_CACHED_DEV_STOP_ALWAYS)

						- bcache_device_stop(d);

					- else if (atomic_read(&dc->has_dirty)

						- 1788                 dc->io_disable = true;
1789                 /* make others know io_disable is true earlier */
1790                 smp_mb();
1791                 bcache_device_stop(d);

					- else { means (BCH_CACHED_STOP_AUTO && no dirty)

						- 1797                 pr_warn("stop_when_cache_set_failed of %s is \"auto\" and cache is clean, keep it alive.\n",
1798                         d->disk->disk_name);

			- 1822                 } else {
1823                         bcache_device_stop(d);
1824                 }

如果不是UNREGISTERING，就直接去stop bcache_device就可以了。

当我们echo 1 > /sys/fs/bcache/XXX/stop 的时候，表示我们想stop整个cache_set，那么我们就把对应的backing也stop掉。
			- continue_at(cl, cache_set_flush, system_wq);

这个时候等待所有的caching 引用结束，也就是writeback结束之后，才会去执行cache_set_flush。

				- 清理gc，journal等等资源。

最后closure_return(cl); 结束了caching 的整个生命周期。

开始执行parent也就是cache_set->cl。 对应的cache_set_free函数。

					- cache_set_free 释放内存相关操作。

- allocation

	- bch_cache_set_alloc

		- cache_set->cl

这是根本的closure，这个closure是cache_set最后一个function： cache_set_free。

1860         closure_init(&c->cl, NULL);
1861         set_closure_fn(&c->cl, cache_set_free, system_wq);
		- cache_set->caching

这是stop或者unregister回去调用的一个closure，

这个closure还有一个目的是控制bcache_device对cache_set的引用。每一个后端的bcache_device都会拿到一个这个closure的reference，当所有的reference结束之后，才会去调用parent closure也就是 cache_set->cl，做cache_set_free。

1863         closure_init(&c->caching, &c->cl);
1864         set_closure_fn(&c->caching, __cache_set_unregister, system_wq);

*XMind - Trial Version*