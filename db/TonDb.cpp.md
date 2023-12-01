<details>


  <summary>

    Source Code
    
  </summary>

```C++

#include "vm/db/TonDb.h"

#include "td/utils/tl_helpers.h"
#include "td/utils/Random.h"

#if TDDB_USE_ROCKSDB
#include "td/db/RocksDb.h"
#endif

namespace vm {

template <class StorerT>
void SmartContractMeta::store(StorerT &storer) const {
  using td::store;
  store(stats.cells_total_count, storer);
  store(stats.cells_total_size, storer);
  store(type, storer);
}
template <class ParserT>
void SmartContractMeta::parse(ParserT &parser) {
  using td::parse;
  parse(stats.cells_total_count, parser);
  parse(stats.cells_total_size, parser);
  parse(type, parser);
}

//
// SmartContractDbImpl
//
Ref<Cell> SmartContractDbImpl::get_root() {
  if (sync_root_with_db_ || !new_root_.is_null()) {
    return new_root_;
  }

  sync_root_with_db();
  return new_root_;
}

void SmartContractDbImpl::set_root(Ref<Cell> new_root) {
  CHECK(new_root.not_null());
  sync_root_with_db();
  if (is_dynamic()) {
    cell_db_->dec(new_root_);
  }
  new_root_ = std::move(new_root);
  if (is_dynamic()) {
    cell_db_->inc(new_root_);
  }
}

SmartContractDbImpl::SmartContractDbImpl(td::Slice hash, std::shared_ptr<KeyValueReader> kv)
    : hash_(hash.str()), kv_(std::move(kv)) {
  cell_db_ = DynamicBagOfCellsDb::create();
}

SmartContractMeta SmartContractDbImpl::get_meta() {
  sync_root_with_db();
  return meta_;
}
td::Status SmartContractDbImpl::validate_meta() {
  if (!is_dynamic()) {
    return td::Status::OK();
  }
  sync_root_with_db();
  TRY_RESULT(in_db, kv_->count({}));
  if (static_cast<td::int64>(in_db) != meta_.stats.cells_total_count + 2) {
    return td::Status::Error(PSLICE() << "Invalid meta " << td::tag("expected_count", in_db)
                                      << td::tag("meta_count", meta_.stats.cells_total_count + 2));
  }
  return td::Status::OK();
}

bool SmartContractDbImpl::is_dynamic() const {
  return meta_.type == SmartContractMeta::Dynamic;
}

bool SmartContractDbImpl::is_root_changed() const {
  return !new_root_.is_null() && (db_root_.is_null() || db_root_->get_hash() != new_root_->get_hash());
}

void SmartContractDbImpl::sync_root_with_db() {
  if (sync_root_with_db_) {
    return;
  }
  std::string root_hash;
  kv_->get("root", root_hash);
  std::string meta_serialized;
  kv_->get("meta", meta_serialized);
  // TODO: proper serialization
  td::unserialize(meta_, meta_serialized).ignore();
  sync_root_with_db_ = true;

  if (root_hash.empty()) {
    meta_.type = SmartContractMeta::Static;
    //meta_.type = SmartContractMeta::Dynamic;
  } else {
    if (is_dynamic()) {
      //FIXME: error handling
      db_root_ = cell_db_->load_cell(root_hash).move_as_ok();
    } else {
      std::string boc_serialized;
      kv_->get("boc", boc_serialized);
      BagOfCells boc;
      //TODO: check error
      boc.deserialize(boc_serialized);
      db_root_ = boc.get_root_cell();
    }
    CHECK(db_root_->get_hash().as_slice() == root_hash);
    new_root_ = db_root_;
  }
}

enum { boc_size = 2000 };
void SmartContractDbImpl::prepare_commit_dynamic(bool force) {
  if (!is_dynamic()) {
    CHECK(force);
    meta_.stats = {};
    cell_db_->inc(new_root_);
  }
  cell_db_->prepare_commit();
  meta_.stats.apply_diff(cell_db_->get_stats_diff());

  if (!force && meta_.stats.cells_total_size < boc_size) {
    //LOG(ERROR) << "DYNAMIC -> BOC";
    return prepare_commit_static(true);
  }
  is_dynamic_commit_ = true;
};

void SmartContractDbImpl::prepare_commit_static(bool force) {
  BagOfCells boc;
  boc.add_root(new_root_);
  boc.import_cells().ensure();  // FIXME
  if (!force && boc.estimate_serialized_size(15) > boc_size) {
    //LOG(ERROR) << "BOC -> DYNAMIC ";
    return prepare_commit_dynamic(true);
  }
  if (is_dynamic()) {
    cell_db_->dec(new_root_);
    cell_db_->prepare_commit();
    // stats is invalid now
  }
  is_dynamic_commit_ = false;
  boc_to_commit_ = boc.serialize_to_string(15);
  meta_.stats = {};
}

void SmartContractDbImpl::prepare_transaction() {
  sync_root_with_db();
  if (!is_root_changed()) {
    return;
  }

  if (is_dynamic()) {
    prepare_commit_dynamic(false);
  } else {
    prepare_commit_static(false);
  }
}

void SmartContractDbImpl::commit_transaction(KeyValue &kv) {
  if (!is_root_changed()) {
    return;
  }

  if (is_dynamic_commit_) {
    //LOG(ERROR) << "STORE DYNAMIC";
    if (!is_dynamic() && db_root_.not_null()) {
      kv.erase("boc");
    }
    CellStorer storer(kv);
    cell_db_->commit(storer);
    meta_.type = SmartContractMeta::Dynamic;
  } else {
    //LOG(ERROR) << "STORE BOC";
    if (is_dynamic() && db_root_.not_null()) {
      //LOG(ERROR) << "Clear Dynamic db";
      CellStorer storer(kv);
      cell_db_->commit(storer);
      cell_db_ = DynamicBagOfCellsDb::create();
    }
    meta_.type = SmartContractMeta::Static;
    kv.set("boc", boc_to_commit_);
    boc_to_commit_ = {};
  }

  kv.set("root", new_root_->get_hash().as_slice());
  kv.set("meta", td::serialize(meta_));
  db_root_ = new_root_;
}

void SmartContractDbImpl::set_reader(std::shared_ptr<KeyValueReader> reader) {
  kv_ = std::move(reader);
  cell_db_->set_loader(std::make_unique<CellLoader>(kv_));
}

//
// TonDbTransactionImpl
//
SmartContractDb TonDbTransactionImpl::begin_smartcontract(td::Slice hash) {
  SmartContractDb res;
  contracts_.apply(hash, [&](auto &info) {
    if (!info.is_inited) {
      info.is_inited = true;
      info.hash = hash.str();
      info.smart_contract_db = std::make_unique<SmartContractDbImpl>(hash, nullptr);
    }
    LOG_CHECK(info.generation_ != generation_) << "Cannot begin one smartcontract twice during the same transaction";
    CHECK(info.smart_contract_db);
    info.smart_contract_db->set_reader(std::make_shared<td::PrefixedKeyValueReader>(reader_, hash));
    res = std::move(info.smart_contract_db);
  });
  return res;
}

void TonDbTransactionImpl::commit_smartcontract(SmartContractDb txn) {
  commit_smartcontract(SmartContractDiff(std::move(txn)));
}
void TonDbTransactionImpl::commit_smartcontract(SmartContractDiff txn) {
  {
    td::PrefixedKeyValue kv(kv_, txn.hash());
    txn.commit_transaction(kv);
  }
  end_smartcontract(txn.extract_smartcontract());
}

void TonDbTransactionImpl::abort_smartcontract(SmartContractDb txn) {
  end_smartcontract(std::move(txn));
}
void TonDbTransactionImpl::abort_smartcontract(SmartContractDiff txn) {
  end_smartcontract(txn.extract_smartcontract());
}

TonDbTransactionImpl::TonDbTransactionImpl(std::shared_ptr<KeyValue> kv) : kv_(std::move(kv)) {
  CHECK(kv_ != nullptr);
  reader_.reset(kv_->snapshot().release());
}

void TonDbTransactionImpl::begin() {
  kv_->begin_write_batch();
  generation_++;
}
void TonDbTransactionImpl::commit() {
  kv_->commit_write_batch();
  reader_.reset(kv_->snapshot().release());
}
void TonDbTransactionImpl::abort() {
  kv_->abort_write_batch();
}
void TonDbTransactionImpl::clear_cache() {
  contracts_ = {};
}

void TonDbTransactionImpl::end_smartcontract(SmartContractDb smart_contract) {
  contracts_.apply(smart_contract->hash(), [&](auto &info) {
    CHECK(info.hash == smart_contract->hash());
    CHECK(!info.smart_contract_db);
    info.smart_contract_db = std::move(smart_contract);
  });
}

//
// TonDbImpl
//
TonDbImpl::TonDbImpl(std::unique_ptr<KeyValue> kv)
    : kv_(std::move(kv)), transaction_(std::make_unique<TonDbTransactionImpl>(kv_)) {
}
TonDbImpl::~TonDbImpl() {
  CHECK(transaction_);
  kv_->flush();
}
TonDbTransaction TonDbImpl::begin_transaction() {
  CHECK(transaction_);
  transaction_->begin();
  return std::move(transaction_);
}
void TonDbImpl::commit_transaction(TonDbTransaction transaction) {
  CHECK(!transaction_);
  CHECK(&transaction->kv() == kv_.get());
  transaction_ = std::move(transaction);
  transaction_->commit();
}
void TonDbImpl::abort_transaction(TonDbTransaction transaction) {
  CHECK(!transaction_);
  CHECK(&transaction->kv() == kv_.get());
  transaction_ = std::move(transaction);
  transaction_->abort();
}
void TonDbImpl::clear_cache() {
  CHECK(transaction_);
  transaction_->clear_cache();
}

std::string TonDbImpl::stats() const {
  return kv_->stats();
}

td::Result<TonDb> TonDbImpl::open(td::Slice path) {
#if TDDB_USE_ROCKSDB
  TRY_RESULT(rocksdb, td::RocksDb::open(path.str()));
  return std::make_unique<TonDbImpl>(std::make_unique<td::RocksDb>(std::move(rocksdb)));
#else
  return td::Status::Error("TonDb is not supported in this build");
#endif
}

}  // namespace vm




```



</details>

This C++ code appears to be part of the implementation for a virtual machine related to smart contract execution in the TON (The Open Network) blockchain. Let's break down the code and provide a short description of various elements:

1. **TonDb.h**:
   - The file includes necessary headers, such as "td/utils/tl_helpers.h" and "td/utils/Random.h."
   - There's a conditional inclusion of "td/db/RocksDb.h" based on the macro `TDDB_USE_ROCKSDB."
   - The code is enclosed in the `vm` namespace.

2. **SmartContractMeta**:
   - A structure or class that seems to represent metadata associated with a smart contract.
   - It has serialization and parsing methods.

3. **SmartContractDbImpl**:
   - Implements the functionality for managing the storage of smart contracts.
   - Defines methods for getting and setting the root of the contract, handling dynamic and static contracts, and preparing commits.
   - Uses a KeyValueReader for reading key-value pairs associated with the contract.
   - Includes methods for handling transactions, committing changes, and setting readers.

4. **TonDbTransactionImpl**:
   - Manages transactions related to TonDb, including smart contract transactions.
   - Begins, commits, or aborts transactions and clears the cache.
   - Handles the beginning and ending of smart contract transactions.

5. **TonDbImpl**:
   - Implements the TonDb interface, which appears to be a database-related component.
   - Uses a KeyValue object for underlying storage.
   - Provides methods for beginning, committing, and aborting transactions, as well as clearing the cache.

6. **Miscellaneous**:
   - There are references to BagOfCells, DynamicBagOfCellsDb, CellStorer, and CellLoader, which likely play a role in managing the storage and serialization of data.
   - The code involves serialization and deserialization of data using td::serialize and td::unserialize.
   - Macros like `CHECK` and `LOG_CHECK` are used for assertions and logging.

7. **Conditional Compilation**:
   - Certain parts of the code are conditionally compiled based on the macro `TDDB_USE_ROCKSDB`.
   - RocksDB-related functionality is included if this macro is defined.

8. **Error Handling**:
   - The code includes some basic error handling using `TRY_RESULT` and `td::Status`.

9. **Comments and TODOs**:
   - There are comments indicating areas for improvement, such as proper serialization and error handling.
   - There's a TODO comment suggesting the need for proper serialization.

Overall, this code seems to be a crucial part of the TON blockchain's implementation, handling the storage and management of smart contracts within a database. The conditional compilation suggests support for different underlying storage engines.
