---
title: "Lab"
date: 2022-06-09T18:25:18+08:00
draft: true
---

## attention

* 注意在每次提交之前格式化下代码，检查下代码语法是否正确，评分系统会检查代码格式和编码规范，不正确直接编译失败
  ```shell
    make format
    make check-lint
    make check-clang-tidy
  ```

## lab0 C++ Primer

  目的是为了熟悉C++的使用，照顾之前没有使用过C++的同学，比较简单的题目  
  * 实现矩阵
  * 实现矩阵的加法乘法以及一个复合运算

  容易出问题的地方
  1. 矩阵使用二维数组，不能直接new，需要先new row，再new col，析构的时候也是需要分步进行，否则会有内存泄漏
  2. 智能指针使用get可以获得原始指针，他们之间不能直接隐式转换
  3. 测试中只有add和mult运算的测试案例，没有复合运算的，需要自己添加测试，否则不确定自己实现是否正确，因为模板有个懒运算的特性，没有使用的模板。是不会检测错误的
     但是评分系统有更全面的测试，所以最好自己简单测试以下

```c++
diff --git a/src/include/primer/p0_starter.h b/src/include/primer/p0_starter.h
index 81fbb56..c87003f 100644
--- a/src/include/primer/p0_starter.h
+++ b/src/include/primer/p0_starter.h
@@ -35,7 +35,7 @@ class Matrix {
    * @param cols The number of columns
    *
    */
-  Matrix(int rows, int cols) {}
+  Matrix(int rows, int cols) : rows_(rows), cols_(cols) {}
 
   /** The number of rows in the matrix */
   int rows_;
@@ -112,19 +112,25 @@ class RowMatrix : public Matrix<T> {
    * @param rows The number of rows
    * @param cols The number of columns
    */
-  RowMatrix(int rows, int cols) : Matrix<T>(rows, cols) {}
+  RowMatrix(int rows, int cols) : Matrix<T>(rows, cols) {
+    data_ = new T *[rows];
+    for (int i = 0; i < rows; i++) {
+      data_[i] = new T[cols];
+      memset(data_[i], 0, sizeof(T) * cols);
+    }
+  }
 
   /**
    * TODO(P0): Add implementation
    * @return The number of rows in the matrix
    */
-  int GetRowCount() const override { return 0; }
+  int GetRowCount() const override { return this->rows_; }
 
   /**
    * TODO(P0): Add implementation
    * @return The number of columns in the matrix
    */
-  int GetColumnCount() const override { return 0; }
+  int GetColumnCount() const override { return this->cols_; }
 
   /**
    * TODO(P0): Add implementation
@@ -139,7 +145,11 @@ class RowMatrix : public Matrix<T> {
    * @throws OUT_OF_RANGE if either index is out of range
    */
   T GetElement(int i, int j) const override {
-    throw NotImplementedException{"RowMatrix::GetElement() not implemented."};
+    if (i < 0 || i >= this->rows_ || j < 0 || j >= this->cols_) {
+      throw Exception(ExceptionType::OUT_OF_RANGE, "RowMatrix::GetElement() out of range");
+    }
+
+    return data_[i][j];
   }
 
   /**
@@ -152,7 +162,13 @@ class RowMatrix : public Matrix<T> {
    * @param val The value to insert
    * @throws OUT_OF_RANGE if either index is out of range
    */
-  void SetElement(int i, int j, T val) override {}
+  void SetElement(int i, int j, T val) override {
+    if (i < 0 || i >= this->rows_ || j < 0 || j >= this->cols_) {
+      throw Exception(ExceptionType::OUT_OF_RANGE, "RowMatrix::GetElement() out of range");
+    }
+
+    data_[i][j] = val;
+  }
 
   /**
    * TODO(P0): Add implementation
@@ -166,7 +182,15 @@ class RowMatrix : public Matrix<T> {
    * @throws OUT_OF_RANGE if `source` is incorrect size
    */
   void FillFrom(const std::vector<T> &source) override {
-    throw NotImplementedException{"RowMatrix::FillFrom() not implemented."};
+    if (static_cast<int>(source.size()) != this->rows_ * this->cols_) {
+      throw Exception(ExceptionType::OUT_OF_RANGE, "RowMatrix::GetElement() out of range");
+    }
+
+    for (int i = 0; i < GetRowCount(); i++) {
+      for (int j = 0; j < GetColumnCount(); j++) {
+        data_[i][j] = source[i * this->cols_ + j];
+      }
+    }
   }
 
   /**
@@ -174,7 +198,12 @@ class RowMatrix : public Matrix<T> {
    *
    * Destroy a RowMatrix instance.
    */
-  ~RowMatrix() override = default;
+  ~RowMatrix() override {
+    for (int i = 0; i < this->rows_; i++) {
+      delete[] data_[i];
+    }
+    delete[] data_;
+  };
 
  private:
   /**
@@ -204,7 +233,17 @@ class RowMatrixOperations {
    */
   static std::unique_ptr<RowMatrix<T>> Add(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB) {
     // TODO(P0): Add implementation
-    return std::unique_ptr<RowMatrix<T>>(nullptr);
+    if (matrixA->GetRowCount() != matrixB->GetRowCount() || matrixA->GetColumnCount() != matrixB->GetColumnCount()) {
+      throw Exception(ExceptionType::OUT_OF_RANGE, "RowMatrix::GetElement() out of range");
+    }
+
+    auto matrix = std::make_unique<RowMatrix<T>>(matrixB->GetRowCount(), matrixB->GetColumnCount());
+    for (int i = 0; i < matrixB->GetRowCount(); i++) {
+      for (int j = 0; j < matrixB->GetColumnCount(); j++) {
+        matrix->SetElement(i, j, matrixA->GetElement(i, j) + matrixB->GetElement(i, j));
+      }
+    }
+    return matrix;
   }
 
   /**
@@ -216,7 +255,21 @@ class RowMatrixOperations {
    */
   static std::unique_ptr<RowMatrix<T>> Multiply(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB) {
     // TODO(P0): Add implementation
-    return std::unique_ptr<RowMatrix<T>>(nullptr);
+    if (matrixA->GetColumnCount() != matrixB->GetRowCount()) {
+      throw Exception(ExceptionType::OUT_OF_RANGE, "RowMatrix::GetElement() out of range");
+    }
+
+    auto matrix = std::make_unique<RowMatrix<T>>(matrixA->GetRowCount(), matrixB->GetColumnCount());
+
+    for (int i = 0; i < matrix->GetRowCount(); i++) {
+      for (int j = 0; j < matrix->GetColumnCount(); j++) {
+        for (int k = 0; k < matrixA->GetColumnCount(); k++) {
+          auto v = matrix->GetElement(i, j);
+          matrix->SetElement(i, j, matrixA->GetElement(i, k) * matrixB->GetElement(k, j) + v);
+        }
+      }
+    }
+    return matrix;
   }
 
   /**
@@ -230,7 +283,13 @@ class RowMatrixOperations {
   static std::unique_ptr<RowMatrix<T>> GEMM(const RowMatrix<T> *matrixA, const RowMatrix<T> *matrixB,
                                             const RowMatrix<T> *matrixC) {
     // TODO(P0): Add implementation
-    return std::unique_ptr<RowMatrix<T>>(nullptr);
+    if (matrixA->GetColumnCount() != matrixB->GetRowCount() || matrixA->GetRowCount() != matrixC->GetRowCount() ||
+        matrixB->GetColumnCount() != matrixC->GetColumnCount()) {
+      throw Exception(ExceptionType::OUT_OF_RANGE, "RowMatrix::GetElement() out of range");
+    }
+
+    auto mult = Add(Multiply(matrixA, matrixB).get(), matrixC);
+    return mult;
   }
 };
 }  // namespace bustub

```


## lab1

共有三个任务，是依赖关系，必须按顺序实现
### LRU REPLACEMENT POLICY

  实现lru替换算法，常见算法，leetcode上有类似的题目，甚至比这个更难一点，这里只需要实现几个简单的函数即可，用于替换frame_id_t

  使用链表map实现，其中链表保存数据，map用于快速查找，这里的代码简单所以大部分人写出来的代码理论上差别不大，自己没有头绪的话可以简单的看下他人的实现，但是不要照抄

头文件实现  
```c++
  std::mutex latch_;
  size_t capacity_;
  std::list<frame_id_t> lru_list_;
  std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> lru_map_;
```

* `bool LRUReplacer::Victim(frame_id_t *frame_id)`  
  选择替换的frame_id_t  
  1. 检查map是否为空，为空则直接返回false
  2. 返回链表的最后一个元素，且把他从链表和map中删除

* `void LRUReplacer::Pin(frame_id_t frame_id)`  
  pin的意思是此时有其他线程使用此frame_id，所以他是不可替换的，也就不需要使用LRUReplacer维护，所以直接从LRUReplacer中删除  
  1. 使用map查找frame_id，如果没有查找到，则直接返回，这里如果直接从list中查找，则可能时间复杂度比较高
  2. 如果有，则从list和map中删除

* `void LRUReplacer::Unpin(frame_id_t frame_id)`  
  Unpin的意思是其他操作不使用此frame_id，可以使用LRUReplacer维护frame_id，等待替换  
  1. 查找frame_id，如果存在，则直接返回
  2. 如果不存在，则检查是否full，如果满了，则删除链表最后的元素，同时更新map
  3. 添加frame_id到链表头部，更新map
  ```c++
  void LRUReplacer::Unpin(frame_id_t frame_id) {
    latch_.lock();

    if (lru_map_.find(frame_id) == lru_map_.end()) {
      while (lru_map_.size() >= capacity_) {
        auto frame = lru_list_.back();
        lru_map_.erase(frame);
        lru_list_.pop_back();
      }

      lru_list_.push_front(frame_id);
      lru_map_[frame_id] = lru_list_.begin();
    }

    latch_.unlock();
  }
  ```

上面的操作全部需要加锁

### BUFFER POOL MANAGER INSTANCE  

  此次实验的核心，维护一个buffer pool，管理磁盘和内存中的页面替换  

  首先`BufferPoolManagerInstance`维护一个page数组，大小为pool_size，数组的下边使用frame_id_t表示，page本身自己有pageid，
  还使用一个`page_table_`维护frame_id_t和page_id_t的映射关系，所以简单的理解就是`BufferPoolManagerInstance`本质还是负责
  page的缓存，但是由于容量有限，所以使用另外的frame_id_t表示缓存中的page，所以外部接口使用page_id_t操作，但是内部使用frame_id_t
  操作page


* `Page *BufferPoolManagerInstance::NewPgImp(page_id_t *page_id)`  
  在缓存中创建一个page，有无输入，有两个返回值，page_id和page，按照他的提示的步骤实现代码
  ```c++
    // 0.   Make sure you call AllocatePage!
    // 1.   If all the pages in the buffer pool are pinned, return nullptr.
    // 2.   Pick a victim page P from either the free list or the replacer. Always pick from the free list first.
    // 3.   Update P's metadata, zero out memory and add P to the page table.
    // 4.   Set the page ID output parameter. Return a pointer to P.
  ```
  * 如果所有的page都是pinned的，则代表缓存容量已满且所有的page都在使用中，无法创建
  * 否则从free_list_或者replacer中选择一个frame_id_t
    1. 先从free_list_查找，他在初始化的时候保存了大小为pool_size的空闲的frame_id_t
    2. 如果没有，则从replacer选择一个页面替换，replacer中能查找的是缓存中无人使用的page
    3. 如果从replacer找到page，且是脏页，则先执行WritePage
    4. 如果都没有找到，则直接null
  * 找到之后，使用`AllocatePage`生成新page且得到pageid，使用此pageid和查找到的frame_id_t更新信息
  * attention  
    1. 使用latch
    2. 返回页面是需要执行`replacer_->Pin(frame_id);`，此时返回的页面是在使用中的
    3. 注意设置参数列表中的page_id，他是一个返回值
  ```c++
    Page *BufferPoolManagerInstance::NewPgImp(page_id_t *page_id) {
      // 0.   Make sure you call AllocatePage!
      // 1.   If all the pages in the buffer pool are pinned, return nullptr.
      // 2.   Pick a victim page P from either the free list or the replacer. Always pick from the free list first.
      // 3.   Update P's metadata, zero out memory and add P to the page table.
      // 4.   Set the page ID output parameter. Return a pointer to P.

      latch_.lock();

      // 1.   If all the pages in the buffer pool are pinned, return nullptr.
      bool allpined = true;
      for (size_t i = 0; i < pool_size_; i++) {
        if (pages_[i].GetPinCount() == 0) {
          allpined = false;
          break;
        }
      }

      if (allpined) {
        latch_.unlock();
        return nullptr;
      }

      // 2.   Pick a victim page P from either the free list or the replacer. Always pick from the free list first.
      frame_id_t frame_id = -1;
      if (!free_list_.empty()) {
        frame_id = free_list_.front();
        free_list_.pop_front();
      } else if (replacer_->Victim(&frame_id)) {
        page_id_t replacepage = -1;
        for (auto &&p : page_table_) {
          if (p.second == frame_id) {
            replacepage = p.first;
            break;
          }
        }

        if (replacepage != -1) {
          auto page = &pages_[frame_id];
          if (page->IsDirty()) {
            disk_manager_->WritePage(replacepage, page->GetData());
            page->pin_count_ = 0;
            page->is_dirty_ = false;
          }
        }
      } else {
        latch_.unlock();
        return nullptr;
      }

     // 3.   Update P's metadata, zero out memory and add P to the page table.
     // 0.   Make sure you call AllocatePage!
     auto pageid = AllocatePage();

     auto page = &pages_[frame_id];
     page_table_.erase(page->page_id_);
     page->pin_count_++;
     page->page_id_ = pageid;
     page->is_dirty_ = false;
     replacer_->Pin(frame_id);
     page_table_[pageid] = frame_id;

     disk_manager_->WritePage(pageid, page->GetData());

     *page_id = pageid;

     latch_.unlock();
     return page;
  }
  ```

* `Page *BufferPoolManagerInstance::FetchPgImp(page_id_t page_id)`  
  * 如果找到page，直接返回
  * 否则尝试替换一个unpin的页面，和NewPgImp中的逻辑类似，从free_list_和replacer查找
  * 不能找到，则返回
  * 能找到，则替换页面信息且此时需要执行`disk_manager_->ReadPage(page_id, page->data_);`操作，设置page中的数据

* `bool BufferPoolManagerInstance::DeletePgImp(page_id_t page_id)`  
  直接按照步骤实现即可

* `bool BufferPoolManagerInstance::UnpinPgImp(page_id_t page_id, bool is_dirty) `   
  UnpinPgImp操作主要是通知pool调用者不再使用此page，此时pin_count_自减1  
  1. 如果GetPinCount已经为0，则直接返回
  2. 否则设置is_dirty
  3. pin_count_--
  4. 最后判断GetPinCount是否为0，如果是，则page无人使用，执行`replacer_->Unpin(frame);`，使用replacer_管理

#### attention

  1. 所有的操作需要使用latch
  2. 不能直接使用其他成员函数，因为都是使用latch的且使用的是同一个latch，所以直接使用会有死锁问题，例如调用`FlushPgImp`

### PARALLEL BUFFER POOL MANAGER
  并发操作buffer pool，buffer pool使用latch，所以并发不会太高，所以这里使用`ParallelBufferPoolManager`维护多个`BufferPoolManagerInstance`
  实现更高的并发操作

  这里使用`page_id mod num_instances`把page_id映射到具体的`BufferPoolManagerInstance`上，且每个`BufferPoolManagerInstance`都有`uint32_t num_instances` and `uint32_t instance_index`所以具体的函数操作只需要简单的调用`BufferPoolManager *ParallelBufferPoolManager::GetBufferPoolManager(page_id_t page_id)`直接操作即可
  ```c++
    BufferPoolManager *ParallelBufferPoolManager::GetBufferPoolManager(page_id_t page_id) {
      // Get BufferPoolManager responsible for handling given page id. You can use this method in your other methods.
      return buffer_pool_manager_instance_[page_id % num_instances_];
    }
  ``


