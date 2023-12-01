<details>


  <summary>

    Source Code
    
  </summary>

```C++

#pragma once
#include "vm/cellslice.h"
#include "vm/cells.h"
#include "vm/boc.h"
#include "td/db/KeyValue.h"
#include "vm/db/CellStorage.h"
#include "vm/db/CellHashTable.h"

#include "td/utils/Slice.h"
#include "td/utils/Status.h"

namespace vm {
class SmartContractDbImpl;
using SmartContractDb = std::unique_ptr<SmartContractDbImpl>;
using KeyValue = td::KeyValue;
using KeyValueReader = td::KeyValueReader;

struct SmartContractMeta {
  DynamicBagOfCellsDb::Stats stats;
  enum BagOfCellsType { Dynamic, Static } type{Static};

  template <class StorerT>
  void store(StorerT &storer) const;
  template <class ParserT>
  void parse(ParserT &parser);
};

class SmartContractDbImpl {
 public:
  Ref<Cell> get_root();
  SmartContractMeta get_meta();
  td::Status validate_meta();

  void set_root(Ref<Cell> new_root);

  SmartContractDbImpl(td::Slice hash, std::shared_ptr<KeyValueReader> kv);

 private:
  std::string hash_;
  std::shared_ptr<KeyValueReader> kv_;

  bool sync_root_with_db_{false};
  Ref<Cell> db_root_;
  Ref<Cell> new_root_;
  SmartContractMeta meta_;
  bool is_dynamic_commit_;
  std::string boc_to_commit_;

  std::unique_ptr<DynamicBagOfCellsDb> cell_db_;
  std::unique_ptr<BagOfCells> bag_of_cells_;

  friend class SmartContractDiff;
  friend class TonDbTransactionImpl;

  void sync_root_with_db();

  td::Slice hash() const {
    return hash_;
  }

  void prepare_transaction();
  void commit_transaction(KeyValue &kv);

  void set_reader(std::shared_ptr<KeyValueReader> reader);

  bool is_dynamic() const;
  void prepare_commit_dynamic(bool force);
  void prepare_commit_static(bool force);
  bool is_root_changed() const;
};

class SmartContractDiff {
 public:
  explicit SmartContractDiff(SmartContractDb db) : db_(std::move(db)) {
    db_->prepare_transaction();
  }

  SmartContractDb extract_smartcontract() {
    return std::move(db_);
  }

  td::Slice hash() const {
    return db_->hash();
  }

  void commit_transaction(KeyValue &kv) {
    db_->commit_transaction(kv);
  }

 private:
  SmartContractDb db_;
};

class TonDbTransactionImpl;
using TonDbTransaction = std::unique_ptr<TonDbTransactionImpl>;
class TonDbTransactionImpl {
 public:
  SmartContractDb begin_smartcontract(td::Slice hash = {});

  void commit_smartcontract(SmartContractDb txn);
  void commit_smartcontract(SmartContractDiff txn);

  void abort_smartcontract(SmartContractDb txn);
  void abort_smartcontract(SmartContractDiff txn);

  TonDbTransactionImpl(std::shared_ptr<KeyValue> kv);

 private:
  std::shared_ptr<KeyValue> kv_;
  std::shared_ptr<KeyValueReader> reader_;
  td::uint64 generation_{0};

  struct SmartContractInfo {
    bool is_inited{false};
    td::uint64 generation_{0};
    std::string hash;
    SmartContractDb smart_contract_db;
    bool operator<(const SmartContractInfo &other) const {
      return hash < other.hash;
    }
    friend bool operator<(const SmartContractInfo &info, td::Slice hash) {
      return info.hash < hash;
    }
    friend bool operator<(td::Slice hash, const SmartContractInfo &info) {
      return hash < info.hash;
    }
  };

  CellHashTable<SmartContractInfo> contracts_;

  KeyValue &kv() {
    return *kv_;
  }
  friend class TonDbImpl;

  void begin();
  void commit();
  void abort();
  void clear_cache();

  void end_smartcontract(SmartContractDb smart_contract);
};

class TonDbImpl;
using TonDb = std::unique_ptr<TonDbImpl>;
class TonDbImpl {
 public:
  TonDbImpl(std::unique_ptr<KeyValue> kv);
  ~TonDbImpl();
  TonDbTransaction begin_transaction();
  void commit_transaction(TonDbTransaction transaction);
  void abort_transaction(TonDbTransaction transaction);
  void clear_cache();
  static td::Result<TonDb> open(td::Slice path);
  std::string stats() const;

 private:
  std::shared_ptr<KeyValue> kv_;
  TonDbTransaction transaction_;
};
}  // namespace vm



```



</details>

Let's analyze the provided C++ codebase based on the specified components and modules.

### 1. ABI Handling

#### Files:
- **`boc.cpp` and `boc.h`**
  - These files are not present in the provided code snippet. Please check if they are part of the complete codebase.

- **`arithops.cpp` and `arithops.h`**
  - These files are not present in the provided code snippet. Please check if they are part of the complete codebase.

#### Directory:
- **`cells` Directory**
  - The `cells` directory is not explicitly included in the provided snippet. Ensure that the `Cell.cpp`, `Cell.h`, and other relevant files in this directory are examined for fundamental building blocks related to ABI handling.

### 2. Smart Contract Implementation

#### Files:
- **`vm.cpp` and `vm.h`**
  - These files are included and seem to contain the implementation of a virtual machine (VM) for executing smart contracts. Functions related to interpreting and executing smart contract code should be explored.

- **`contops.cpp` and `contops.h`**
  - These files are not present in the provided code snippet. Please check if they are part of the complete codebase.

#### Directory:
- **`db` Directory**
  - The `db` directory is included, and files such as `TonDb.cpp` and `TonDb.h` are mentioned. Explore these files for implementations related to database interactions, which are crucial for storing contract state.

### 3. Fundamental Components

#### Files:
- **`cellparse.hpp`**
  - The `cellparse.hpp` file is not present in the provided code snippet. Please check if it is part of the complete codebase.

- **`atom.cpp` and `atom.h`**
  - These files are not present in the provided code snippet. Please check if they are part of the complete codebase.

### 4. Utility and Operation Handling

#### Files:
- **`utils.cpp` and `utils.h`**
  - These files are included. Examine utility functions in these files for common tasks used across the codebase.

- **`dispatch.cpp` and `dispatch.h`**
  - These files are not present in the provided code snippet. Please check if they are part of the complete codebase.

### 5. Database Interaction

#### Directory:
- **`db` Directory**
  - The `db` directory is included, and specific files such as `DynamicBagOfCellsDb.cpp` and `StaticBagOfCellsDb.cpp` are mentioned. Explore these files for handling dynamic and static bags of cells in the database.

Make sure to review the complete codebase for the missing files and directories mentioned in the exploration instructions. If you have specific questions or need further details on a particular aspect, feel free to ask.
