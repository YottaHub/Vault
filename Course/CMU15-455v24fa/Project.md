# P1: LRU-K Buffer Pool
## LRU-K Replacer Data Structure
1. Separate evict-able and in-evict-able nodes into two `unordered_map<frame_id_t, NodePtr>` for $O(1)$ time complexity search
	1. `using NodePtr = std::shared_ptr<LRUKNode>;` for memory management
2. `Evict()` works for evict-able nodes only. Use `map<size_t, NodePtr>` for $O(log(n))$ time complexity victim decision.
	1. With limited space complexity $O(n)$, where $n$ is the number of nodes in the replacer, $O(1)$ victim decision is not possible
	2. Lock strategy: 
	3. // AccessType::Scan, most-recent-use  
		// AccessType::LookUp, zipf distribution  
		// AccessType::Index, force lru-k
1. `RecordAccess` only access in-evict-able frames
```cpp
class LRUKNode {  
  friend class LRUKReplacer;  
  
 public:  
  explicit LRUKNode(frame_id_t fid);  
  
 private:  
  /** @brief History of last seen K timestamps of this page. Least recent timestamp stored in front. */  
  std::list<size_t> history_;  
  frame_id_t fid_;  
  bool is_evictable_{false};  
  [[maybe_unused]] bool is_lru_k_node_{false};  
};

/** @brief All stored nodes **/  
std::unordered_map<frame_id_t, NodePtr> node_store_;  
/** @brief LRU eviction nodes **/  
std::map<size_t, NodePtr> lru_node_store_;  
/** @brief K-distance eviction nodes **/  
std::map<size_t, NodePtr> lru_k_node_store_;
```

```cpp
class LRUKNode {  
  friend class LRUKReplacer;  
  
 public:  
  explicit LRUKNode(frame_id_t fid);  
  
 private:  
  /** @brief History of last seen K timestamps of this page. Least recent timestamp stored in front. */  
  std::list<size_t> history_;  
  frame_id_t fid_;  
  bool is_evictable_{false};  
  bool is_lru_k_node_{false};
  LRUKNode* next_lru_node;
  LRUKNode* next_lru_k_node;
  LRUKNode* prev_lru_node;
  LRUKNode* prev_lru_k_node;
};

/** @brief All stored nodes **/  
std::unordered_map<frame_id_t, NodePtr> node_store_;
NodePtr dummy_lru_;
NodePtr dummy_lru_k_;
```

P2: Database Index
```
/autograder/source/bustub/src/storage/index/b_plus_tree.cpp:254:29: note: the use and move are unsequenced, i.e. there is no guarantee about the order in which they are evaluated
  return INDEXITERATOR_TYPE(guard.GetPageId(), leaf->KeyIndex(key, comparator_), bpm_, std::move(guard));                                                              
```

重写11.12-11.13的LeetCode Daily + Basic Binary Search w/ conditions plus `KeyIndex` in internal and leaf page + Remove with distribution


```cpp
if (auto it = page_table_.find(page_id); it != page_table_.end()) {  
  auto frame = frames_[it->second];  
  while (frame->page_id_ != page_id) {  
    frame->cv_->wait(guard, [&] { return frame->page_id_ == page_id; });  
  }  
  replacer_->RecordAccess(it->second, access_type);  
  frame->pin_count_.fetch_add(1);  
  replacer_->SetEvictable(it->second, false);  
  return it->second;  
}

page_table_[page_id] = frame_id.value();

void BufferPoolManager::DiskRequestAsync(page_id_t page_id, const std::shared_ptr<FrameHeader> &frame,  
                                         std::unique_lock<std::mutex> &lock, bool is_write) {  
  auto promise = disk_scheduler_->CreatePromise();  
  auto future = promise.get_future();  
  DiskRequest request{  
      .is_write_ = is_write,  
      .data_ = frame->data_.data(),  
      .page_id_ = page_id,  
      .callback_ = std::move(promise),  
      .cv_ = frame->cv_,  
  };  
  disk_scheduler_->Schedule(std::move(request));  
  future.get();  
}

void DiskScheduler::StartWorkerThread() {  
  std::optional<DiskRequest> r;  
  while (true) {  
    r = request_queue_.Get();  
    if (!r.has_value()) {  
      break;  
    }  
  
    RequestAsync(std::move(r.value()));  
  }  
}  
  
void DiskScheduler::RequestAsync(DiskRequest r) {   
  if (r.is_write_) {  
    disk_manager_->WritePage(r.page_id_, r.data_);  
  } else {  
    disk_manager_->ReadPage(r.page_id_, r.data_);  
  }  
  r.callback_.set_value(true);  
}

```

split a leaf node when the number of values reaches `max_size` after insertion, and split an internal node when number of values reaches `max_size` before insertion. This will ensure that an insertion to a leaf node will never cause a page data overflow when you do something like `InsertIntoLeaf` and then redistribute;**it will also prevent an internal node with only one child**. However, you indeed can do something like lazily splitting leaf node / handling the case for internal node of only one children. It is up to you; our test cases do not test for these conditions.
- Try to check every page's pin_count before split and coalesce
# Project #3: Query Execution 

> [!important] debug on listen to program nc-shell

## Leaderboard Query 1: Too Many Joins!

```sql
explain select count(*), max(__mock_t4_1m.x), max(__mock_t4_1m.y), max(__mock_t5_1m.x), max(__mock_t5_1m.y), max(__mock_t6_1m.x), max(__mock_t6_1m.y)  
    from __mock_t4_1m, __mock_t5_1m, __mock_t6_1m  
        where (__mock_t4_1m.x = __mock_t5_1m.x)  
            and (__mock_t6_1m.y = __mock_t5_1m.y)  
            and (__mock_t4_1m.y >= 1000000)  
            and (__mock_t4_1m.y < 1500000)  
            and (__mock_t6_1m.x < 150000)  
            and (__mock_t6_1m.x >= 100000);
=== OPTIMIZER ===
Agg { types=["count_star", "max", "max", "max", "max", "max", "max"], aggregates=["1", "#0.0", "#0.1", "#0.2", "#0.3", "#0.4", "#0.5"], group_by=[] }
  NestedLoopJoin { type=Inner, predicate=((((((#0.0=#0.2)and(#1.1=#0.3))and(#0.1>=1000000))and(#0.1<1500000))and(#1.0<150000))and(#1.0>=100000)) }
    NestedLoopJoin { type=Inner, predicate=true }
      MockScan { table=__mock_t4_1m }
      MockScan { table=__mock_t5_1m }
    MockScan { table=__mock_t6_1m }
```

**Recommended Optimizations:** Decompose the filter condition to extract hash join keys, and push down the remaining filter conditions to be below the hash join.

- Extract hash join keys so that rule `OptimizeNLJAsHashJoin` works;
- Push down predicates as `FilterPlanNode` over `MockScanPlanNode` (`MockScanNode` does not support integrated predicate)
- [ ] Reorder join order (550000 + 50000 * r * 500000) vs (5500000 + 500000  * t * 50000). I think it is a bad idea, try construct a new three table hash join plan node.
- [ ] Update tuple index while extract hash join keys


```sql
Agg { types=["count_star", "max", "max", "max", "max", "max", "max"], aggregates=["1", "#0.0", "#0.1", "#0.2", "#0.3", "#0.4", "#0.5"], group_by=[] }
  HashJoin { type=Inner, left_key=["#0.3"], right_key=["#1.1"] }
    HashJoin { type=Inner, left_key=["#0.0"], right_key=["#1.0"] }
      Filter { predicate=((#0.1>=1000000)and(#0.1<1500000)) }
        MockScan { table=__mock_t4_1m }
      MockScan { table=__mock_t5_1m }
    Filter { predicate=((#1.0<150000)and(#1.0>=100000)) }
      MockScan { table=__mock_t6_1m }
```

```text
Before Optimization Score: TIMEOUT(gradescope) 
After  Optimizatoin Score: ~2000  (gradescope) ~1000 (M2 mba)
```

## Leaderboard Query 2: The Mad Data Scientist

```sql
explain (o) select v, d1, d2 from (  
    select  
        v, max(v1) as d1, max(v1) + max(v1) + max(v2) as d2,  
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2), 
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2), 
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2), 
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2),
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2), 
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2), 
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2), 
        min(v1), max(v2), min(v2), 
        max(v1) + min(v1), max(v2) + min(v2)  
    from __mock_t7 left join (
	    select v4 from __mock_t8 where 1 == 2
	) on v < v4 
	group by v  
);
```

**Recommended Optimizations:**

- Column pruning – you only need to compute v, d1, d2 from the left table in aggregation. We've provided a skeleton file `column_pruning.cpp`.
- Common expression elimination, transform always false filter to dummy scan (values plan node of zero rows)

```sql
=== OPTIMIZER ===
Projection { exprs=["#0.0", "#0.1", "#0.2"] }
  Projection { exprs=["#0.0", "#0.1", "((#0.2+#0.3)+#0.4)"] }
    Agg { types=["max", "max", "max", "max"], aggregates=["#0.1", "#0.1", "#0.1", "#0.2"], group_by=["#0.0"] }
      NestedLoopJoin { type=Left, predicate=(#0.0<#1.0) }
        MockScan { table=__mock_t7 }
        Values { rows=0 }
```

- As right plan node of the above result is empty, the predicate nested loop join always return false. Meanwhile, the aggregate plan node never touch tuples from the false eliminated table. Thus, we can treat them as a single `MockScanPlanNode`. Also, the two projections at the top are identical, merge them as one by re-using existing `OptimizeMergeProction` rule.


```sql
=== OPTIMIZER ===
Projection { exprs=["#0.0", "#0.1", "((#0.2+#0.3)+#0.4)"] }
  Agg { types=["max", "max", "max", "max"], aggregates=["#0.1", "#0.1", "#0.1", "#0.2"], group_by=["#0.0"] }
    MockScan { table=__mock_t7 }
```

```text
Before Optimization   Score: ~3800(gradescope)
After  Optimizatoin 1 Score: ~750 (gradescope)
After  Optimizatoin 2 Score: ~510 (gradescope) ~380 (M2 mba) 
```

## Leaderboard Query 3: All that work, for nothing?

**Recommended Optimizations:** Bloom filter for the hash table used during the build phase of the hash join.

- The trick is: run left child executor twice, first for building bloom filter and second for build hash join table. Building and check from bloom filter is way faster as long as you make sure that no false positive occurs.

```text
Before Optimization Score: ~550 (gradescope)
After  Optimizatoin Score: ~185 (gradescope) ~135 (M2 mba) 
```

## P3: Query Execution
- for `IndexScanPlanNode`, support greater or equal than, greater than, less than by index iterator, and update `predicate` into an array. [leaderboard test 1]
- landmine in aggregation_executor, if no group by keys then empty table return initial value; if has group by key then empty table return nothing.
- You will want to fetch the tuple from the outer table, construct the index probe key by using `key_predicate`, and then look up the RID in the index to retrieve the corresponding tuple for the inner table. <=> Secondary Index

# Project #4: Concurrency Control (MVCC)

## Bonus Task #2 Serializable Verification

> OCC cover-up: [lecture](https://www.youtube.com/watch?v=-TA0QIwFUnU&list=PLSE8ODhjZXjYDBpQnSymaectKjxCy6BYq&index=20) w/ [[Courses/CMU15-455v24fa/Slides/18-timestampordering.pdf|slides]] + [spec](https://15445.courses.cs.cmu.edu/fall2024/project4/#bonus2) + [[Courses/CMU15-455v24fa/Notes/18-timestampordering.pdf|note]]

### Validation Phase

The project uses OCC **backward validation**.
- Check whether the committing transaction intersects its read/write sets with those of any transaction that have ***already*** committed.

If TS$(T_i)$ $\lt$ TS$(T_j )$, then one of the following three conditions must hold: 
1. $T_i$ completes all three phases before $Tj$ begins its execution (serial ordering). 
2. $Ti$ completes its Write phase before $T_j$ starts its Write phase, and Ti does not write to any object read by $T_j$. 
	- WriteSet$(T_i)\space ∩ \space$ReadSet$(T_j ) = ∅$.
3. $T_i$ completes its Read phase before $T_j$ completes its Read phase, and $T_i$ does not write to any object that is either read or written by $T_j$.
	- $WriteSet(T_i) ∩ ReadSet(T_j ) = ∅$, and
	- $WriteSet(T_i) ∩ WriteSet(T_j ) = empty$.

The project solves phantom read using a similar approach to re-execute scans.