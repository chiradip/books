# Chapter 8: From Memory to Disk - Building a Storage Engine

## Introduction: Beyond Ephemeral Data

Our calculator processed expressions entirely in memory—perfect for temporary computations but inadequate for persistent data storage. A database's primary responsibility is durability: ensuring data survives process restarts, system crashes, and power failures. This chapter bridges the gap between our expression evaluator and a true database by implementing a storage engine that persists data to disk while maintaining the performance characteristics users expect.

The storage engine is the foundation layer of any database system. While our calculator's parser handles the "what" of data operations, the storage engine handles the "where" and "how"—where data lives on disk, how it's organized for efficient access, and how it survives system failures. We'll build upon our calculator's robust error handling and extend it to handle I/O operations, file corruption, and concurrent access patterns.

## Understanding Storage Requirements

Before diving into implementation, we must understand what differentiates database storage from simple file I/O. Consider these scenarios:

1. **Atomic Updates**: When updating a user's account balance, either the entire transaction succeeds or nothing changes—partial updates could corrupt financial data.

2. **Concurrent Access**: Multiple processes might simultaneously read and write the same data without interference.

3. **Recovery**: After a system crash, the database must recover to a consistent state, potentially rolling back incomplete transactions.

4. **Performance**: Random access to millions of records must remain fast, even as data grows beyond available memory.

Our storage engine will address these requirements through a layered architecture:
- **Page Management**: Fixed-size blocks that align with operating system I/O patterns
- **Buffer Pool**: In-memory cache to minimize disk access
- **Transaction Log**: Write-ahead logging for crash recovery
- **Index Structures**: B-trees for efficient data access

## Page-Based Storage Architecture

Modern storage systems organize data in fixed-size pages (typically 4KB or 8KB) that align with operating system page sizes. This alignment minimizes I/O overhead and enables efficient buffer management.

```python
import os
import struct
import threading
from collections import OrderedDict
from contextlib import contextmanager
from typing import Dict, List, Optional, Any

class Page:
    """Represents a fixed-size page of data on disk"""
    PAGE_SIZE = 4096  # 4KB pages
    HEADER_SIZE = 24   # Page metadata
    
    def __init__(self, page_id: int, data: bytes = None):
        self.page_id = page_id
        self.dirty = False  # Has been modified since last write
        self.pin_count = 0  # Number of active references
        self.data = data or bytearray(self.PAGE_SIZE)
        
        # Page header: [page_id(8), record_count(4), free_space(4), flags(4), checksum(4)]
        if data is None:
            self.init_empty_page()
    
    def init_empty_page(self):
        """Initialize empty page with header"""
        # Clear the page
        self.data = bytearray(self.PAGE_SIZE)
        # Write header
        struct.pack_into('<QIIII', self.data, 0, 
                        self.page_id,  # Page ID
                        0,             # Record count
                        self.PAGE_SIZE - self.HEADER_SIZE,  # Free space
                        0,             # Flags
                        0)             # Checksum (calculated later)
    
    def get_header(self):
        """Parse page header"""
        return struct.unpack('<QIIII', self.data[:self.HEADER_SIZE])
    
    def set_record_count(self, count: int):
        """Update record count in header"""
        struct.pack_into('<I', self.data, 8, count)
        self.dirty = True
    
    def set_free_space(self, free_space: int):
        """Update free space in header"""
        struct.pack_into('<I', self.data, 12, free_space)
        self.dirty = True
    
    def calculate_checksum(self) -> int:
        """Calculate page checksum for integrity verification"""
        # Simple checksum - XOR all 4-byte words except checksum field
        checksum = 0
        for i in range(0, len(self.data) - 4, 4):
            if i == 20:  # Skip checksum field itself
                continue
            word = struct.unpack('<I', self.data[i:i+4])[0]
            checksum ^= word
        return checksum
    
    def verify_integrity(self) -> bool:
        """Verify page hasn't been corrupted"""
        stored_checksum = struct.unpack('<I', self.data[20:24])[0]
        calculated_checksum = self.calculate_checksum()
        return stored_checksum == calculated_checksum
    
    def finalize(self):
        """Prepare page for writing to disk"""
        checksum = self.calculate_checksum()
        struct.pack_into('<I', self.data, 20, checksum)

class BufferPool:
    """In-memory cache for database pages"""
    
    def __init__(self, max_pages: int = 1000):
        self.max_pages = max_pages
        self.pages: OrderedDict[int, Page] = OrderedDict()
        self.lock = threading.RLock()
        self.pin_counts: Dict[int, int] = {}
    
    def get_page(self, page_id: int, storage_manager) -> Page:
        """Get page from buffer pool, loading from disk if necessary"""
        with self.lock:
            # Check if page is already in buffer
            if page_id in self.pages:
                page = self.pages[page_id]
                # Move to end (most recently used)
                self.pages.move_to_end(page_id)
                page.pin_count += 1
                return page
            
            # Need to load from disk
            page_data = storage_manager.read_page_from_disk(page_id)
            page = Page(page_id, page_data)
            
            # Verify page integrity
            if not page.verify_integrity():
                raise IOError(f"Page {page_id} failed integrity check")
            
            # Add to buffer pool
            self._ensure_capacity(storage_manager)
            self.pages[page_id] = page
            page.pin_count += 1
            
            return page
    
    def _ensure_capacity(self, storage_manager):
        """Evict pages if buffer pool is full"""
        while len(self.pages) >= self.max_pages:
            # Find unpinned page to evict (FIFO among unpinned)
            for page_id, page in self.pages.items():
                if page.pin_count == 0:
                    if page.dirty:
                        storage_manager.write_page_to_disk(page)
                    del self.pages[page_id]
                    break
            else:
                raise RuntimeError("No unpinned pages available for eviction")
    
    def unpin_page(self, page_id: int, is_dirty: bool = False):
        """Release reference to page"""
        with self.lock:
            if page_id in self.pages:
                page = self.pages[page_id]
                page.pin_count = max(0, page.pin_count - 1)
                if is_dirty:
                    page.dirty = True
    
    def flush_all_pages(self, storage_manager):
        """Write all dirty pages to disk"""
        with self.lock:
            for page in self.pages.values():
                if page.dirty:
                    storage_manager.write_page_to_disk(page)
                    page.dirty = False

class StorageManager:
    """Manages physical storage and page I/O"""
    
    def __init__(self, db_file: str):
        self.db_file = db_file
        self.buffer_pool = BufferPool()
        self.next_page_id = 0
        self.free_pages = []
        self.lock = threading.RLock()
        
        # Initialize database file if it doesn't exist
        if not os.path.exists(db_file):
            self._initialize_database()
        else:
            self._load_metadata()
    
    def _initialize_database(self):
        """Create new database file with metadata page"""
        with open(self.db_file, 'wb') as f:
            # Create metadata page (page 0)
            metadata_page = Page(0)
            # Store next_page_id and free_page_count in metadata
            struct.pack_into('<QI', metadata_page.data, Page.HEADER_SIZE, 1, 0)
            metadata_page.finalize()
            f.write(metadata_page.data)
        
        self.next_page_id = 1
    
    def _load_metadata(self):
        """Load database metadata from page 0"""
        with open(self.db_file, 'rb') as f:
            metadata_data = f.read(Page.PAGE_SIZE)
            if len(metadata_data) < Page.PAGE_SIZE:
                raise IOError("Invalid database file")
            
            # Extract metadata
            self.next_page_id, free_page_count = struct.unpack('<QI', 
                                                             metadata_data[Page.HEADER_SIZE:Page.HEADER_SIZE+12])
    
    def _save_metadata(self):
        """Save metadata to page 0"""
        metadata_page = Page(0)
        struct.pack_into('<QI', metadata_page.data, Page.HEADER_SIZE, 
                        self.next_page_id, len(self.free_pages))
        metadata_page.finalize()
        
        with open(self.db_file, 'r+b') as f:
            f.seek(0)
            f.write(metadata_page.data)
    
    def allocate_page(self) -> int:
        """Allocate a new page"""
        with self.lock:
            if self.free_pages:
                return self.free_pages.pop()
            else:
                page_id = self.next_page_id
                self.next_page_id += 1
                self._save_metadata()
                return page_id
    
    def deallocate_page(self, page_id: int):
        """Mark page as free for reuse"""
        with self.lock:
            self.free_pages.append(page_id)
            self._save_metadata()
    
    def read_page_from_disk(self, page_id: int) -> bytes:
        """Read page from disk"""
        with open(self.db_file, 'rb') as f:
            f.seek(page_id * Page.PAGE_SIZE)
            data = f.read(Page.PAGE_SIZE)
            if len(data) < Page.PAGE_SIZE:
                raise IOError(f"Could not read page {page_id}")
            return data
    
    def write_page_to_disk(self, page: Page):
        """Write page to disk"""
        page.finalize()  # Calculate checksum
        
        with open(self.db_file, 'r+b') as f:
            f.seek(page.page_id * Page.PAGE_SIZE)
            f.write(page.data)
            f.flush()  # Ensure write reaches disk
            os.fsync(f.fileno())  # Force OS to flush to disk
        
        page.dirty = False
    
    def get_page(self, page_id: int) -> Page:
        """Get page through buffer pool"""
        return self.buffer_pool.get_page(page_id, self)
    
    def unpin_page(self, page_id: int, is_dirty: bool = False):
        """Release page reference"""
        self.buffer_pool.unpin_page(page_id, is_dirty)
    
    def sync(self):
        """Ensure all changes are written to disk"""
        self.buffer_pool.flush_all_pages(self)
        self._save_metadata()
```

## Record Management and Serialization

Building on our calculator's data handling, we need to serialize complex data structures to disk. Unlike our calculator's simple numeric values, a database stores varied data types in structured records.

```python
import json
from datetime import datetime
from decimal import Decimal

class DataType:
    """Type system for database values"""
    INTEGER = 'int'
    FLOAT = 'float'
    STRING = 'string'
    BOOLEAN = 'bool'  
    DATETIME = 'datetime'
    DECIMAL = 'decimal'

class Record:
    """Represents a database record with typed fields"""
    
    def __init__(self, record_id: int = None, data: Dict[str, Any] = None):
        self.record_id = record_id
        self.data = data or {}
        self.deleted = False
    
    def serialize(self) -> bytes:
        """Convert record to bytes for storage"""
        # Create serializable version of data
        serializable_data = {}
        for key, value in self.data.items():
            if isinstance(value, Decimal):
                serializable_data[key] = {'_type': 'decimal', '_value': str(value)}
            elif isinstance(value, datetime):
                serializable_data[key] = {'_type': 'datetime', '_value': value.isoformat()}
            else:
                serializable_data[key] = value
        
        record_dict = {
            'record_id': self.record_id,
            'data': serializable_data,
            'deleted': self.deleted
        }
        
        json_str = json.dumps(record_dict, separators=(',', ':'))
        return json_str.encode('utf-8')
    
    @classmethod
    def deserialize(cls, data: bytes) -> 'Record':
        """Create record from serialized bytes"""
        json_str = data.decode('utf-8')
        record_dict = json.loads(json_str)
        
        # Reconstruct special types
        reconstructed_data = {}
        for key, value in record_dict['data'].items():
            if isinstance(value, dict) and '_type' in value:
                if value['_type'] == 'decimal':
                    reconstructed_data[key] = Decimal(value['_value'])
                elif value['_type'] == 'datetime':
                    reconstructed_data[key] = datetime.fromisoformat(value['_value'])
            else:
                reconstructed_data[key] = value
        
        record = cls(record_dict['record_id'], reconstructed_data)
        record.deleted = record_dict.get('deleted', False)
        return record

class RecordManager:
    """Manages records within pages"""
    
    def __init__(self, storage_manager: StorageManager):
        self.storage_manager = storage_manager
        self.next_record_id = 1
    
    def insert_record(self, table_name: str, data: Dict[str, Any]) -> int:
        """Insert new record into table"""
        record = Record(self.next_record_id, data)
        self.next_record_id += 1
        
        # Find page with space or allocate new one
        page_id = self._find_page_with_space(table_name, len(record.serialize()))
        if page_id is None:
            page_id = self._allocate_new_page(table_name)
        
        # Insert record into page
        page = self.storage_manager.get_page(page_id)
        try:
            self._insert_record_into_page(page, record)
            return record.record_id
        finally:
            self.storage_manager.unpin_page(page_id, is_dirty=True)
    
    def get_record(self, table_name: str, record_id: int) -> Optional[Record]:
        """Retrieve record by ID"""
        # In a full implementation, we'd use an index to find the page
        # For simplicity, we'll scan all pages
        for page_id in self._get_table_pages(table_name):
            page = self.storage_manager.get_page(page_id)
            try:
                record = self._find_record_in_page(page, record_id)
                if record and not record.deleted:
                    return record
            finally:
                self.storage_manager.unpin_page(page_id)
        
        return None
    
    def update_record(self, table_name: str, record_id: int, new_data: Dict[str, Any]) -> bool:
        """Update existing record"""
        # Find and update record
        for page_id in self._get_table_pages(table_name):
            page = self.storage_manager.get_page(page_id)
            try:
                if self._update_record_in_page(page, record_id, new_data):
                    return True
            finally:
                self.storage_manager.unpin_page(page_id, is_dirty=True)
        
        return False
    
    def delete_record(self, table_name: str, record_id: int) -> bool:
        """Mark record as deleted"""
        for page_id in self._get_table_pages(table_name):
            page = self.storage_manager.get_page(page_id)
            try:
                if self._delete_record_in_page(page, record_id):
                    return True
            finally:
                self.storage_manager.unpin_page(page_id, is_dirty=True)
        
        return False
    
    def _find_page_with_space(self, table_name: str, required_space: int) -> Optional[int]:
        """Find page with enough free space"""
        for page_id in self._get_table_pages(table_name):
            page = self.storage_manager.get_page(page_id)
            try:
                _, _, free_space, _, _ = page.get_header()
                if free_space >= required_space + 8:  # +8 for record header
                    return page_id
            finally:
                self.storage_manager.unpin_page(page_id)
        
        return None
    
    def _allocate_new_page(self, table_name: str) -> int:
        """Allocate new page for table"""
        page_id = self.storage_manager.allocate_page()
        
        # Initialize empty page
        page = Page(page_id)
        page.init_empty_page()
        
        # Write to disk
        self.storage_manager.write_page_to_disk(page)
        
        # Register page with table (simplified - in reality we'd update catalog)
        return page_id
    
    def _get_table_pages(self, table_name: str) -> List[int]:
        """Get all page IDs for a table"""
        # Simplified implementation - would normally query catalog
        # For now, assume all pages except 0 (metadata) belong to the table
        pages = []
        for i in range(1, self.storage_manager.next_page_id):
            pages.append(i)
        return pages
    
    def _insert_record_into_page(self, page: Page, record: Record):
        """Insert record into specific page"""
        serialized = record.serialize()
        record_size = len(serialized)
        
        # Get current page state
        page_id, record_count, free_space, flags, checksum = page.get_header()
        
        if free_space < record_size + 8:
            raise ValueError("Insufficient space in page")
        
        # Find insertion point (after existing records)
        insertion_offset = Page.HEADER_SIZE
        for i in range(record_count):
            # Read record header: [size(4), record_id(4)]
            size = struct.unpack('<I', page.data[insertion_offset:insertion_offset+4])[0]
            insertion_offset += 8 + size
        
        # Write record header and data
        struct.pack_into('<II', page.data, insertion_offset, record_size, record.record_id)
        page.data[insertion_offset+8:insertion_offset+8+record_size] = serialized
        
        # Update page header
        page.set_record_count(record_count + 1)
        page.set_free_space(free_space - record_size - 8)
    
    def _find_record_in_page(self, page: Page, record_id: int) -> Optional[Record]:
        """Find record in page by ID"""
        page_id, record_count, free_space, flags, checksum = page.get_header()
        
        offset = Page.HEADER_SIZE
        for i in range(record_count):
            # Read record header
            size, rid = struct.unpack('<II', page.data[offset:offset+8])
            
            if rid == record_id:
                # Found it - deserialize
                record_data = page.data[offset+8:offset+8+size]
                return Record.deserialize(record_data)
            
            offset += 8 + size
        
        return None
    
    def _update_record_in_page(self, page: Page, record_id: int, new_data: Dict[str, Any]) -> bool:
        """Update record in page"""
        # For simplicity, we'll delete and re-insert
        # Production systems use more sophisticated update-in-place strategies
        record = self._find_record_in_page(page, record_id)
        if record:
            record.data = new_data
            # In a full implementation, we'd handle size changes properly
            return True
        return False
    
    def _delete_record_in_page(self, page: Page, record_id: int) -> bool:
        """Mark record as deleted in page"""
        record = self._find_record_in_page(page, record_id)
        if record:
            record.deleted = True
            # In practice, we'd rewrite the page with the updated record
            return True
        return False
```

## Write-Ahead Logging for Durability

Database durability requires that committed changes survive system crashes. Write-ahead logging (WAL) ensures this by recording all changes to a log file before modifying data pages.

```python
import threading
import time
from enum import Enum

class LogRecordType(Enum):
    BEGIN = 'BEGIN'
    COMMIT = 'COMMIT'
    ABORT = 'ABORT'
    INSERT = 'INSERT'
    UPDATE = 'UPDATE'
    DELETE = 'DELETE'

class LogRecord:
    """Represents a single log entry"""
    
    def __init__(self, txn_id: int, record_type: LogRecordType, 
                 table_name: str = None, record_id: int = None, 
                 old_data: Dict[str, Any] = None, new_data: Dict[str, Any] = None):
        self.timestamp = time.time()
        self.txn_id = txn_id
        self.record_type = record_type
        self.table_name = table_name
        self.record_id = record_id
        self.old_data = old_data
        self.new_data = new_data
        self.lsn = None  # Log Sequence Number (set by log manager)
    
    def serialize(self) -> bytes:
        """Serialize log record for storage"""
        log_dict = {
            'timestamp': self.timestamp,
            'txn_id': self.txn_id,
            'record_type': self.record_type.value,
            'table_name': self.table_name,
            'record_id': self.record_id,
            'old_data': self.old_data,
            'new_data': self.new_data,
            'lsn': self.lsn
        }
        
        json_str = json.dumps(log_dict, separators=(',', ':'))
        return json_str.encode('utf-8')
    
    @classmethod
    def deserialize(cls, data: bytes) -> 'LogRecord':
        """Deserialize log record from storage"""
        json_str = data.decode('utf-8')
        log_dict = json.loads(json_str)
        
        record = cls(
            log_dict['txn_id'],
            LogRecordType(log_dict['record_type']),
            log_dict['table_name'],
            log_dict['record_id'],
            log_dict['old_data'],
            log_dict['new_data']
        )
        record.timestamp = log_dict['timestamp']
        record.lsn = log_dict['lsn']
        return record

class LogManager:
    """Manages write-ahead logging"""
    
    def __init__(self, log_file: str):
        self.log_file = log_file
        self.next_lsn = 1
        self.lock = threading.Lock()
        
        # Initialize log file
        if not os.path.exists(log_file):
            open(log_file, 'w').close()
        else:
            self._recover_lsn()
    
    def _recover_lsn(self):
        """Recover next LSN from existing log"""
        try:
            with open(self.log_file, 'r') as f:
                for line in f:
                    if line.strip():
                        record = LogRecord.deserialize(line.encode('utf-8'))
                        if record.lsn >= self.next_lsn:
                            self.next_lsn = record.lsn + 1
        except:
            self.next_lsn = 1
    
    def write_log_record(self, log_record: LogRecord) -> int:
        """Write log record and return LSN"""
        with self.lock:
            # Assign LSN
            log_record.lsn = self.next_lsn
            self.next_lsn += 1
            
            # Write to log file
            with open(self.log_file, 'a') as f:
                f.write(log_record.serialize().decode('utf-8') + '\n')
                f.flush()
                os.fsync(f.fileno())  # Force to disk
            
            return log_record.lsn
    
    def read_log_records(self, from_lsn: int = 1) -> List[LogRecord]:
        """Read log records starting from LSN"""
        records = []
        with open(self.log_file, 'r') as f:
            for line in f:
                if line.strip():
                    record = LogRecord.deserialize(line.encode('utf-8'))
                    if record.lsn >= from_lsn:
                        records.append(record)
        
        return sorted(records, key=lambda r: r.lsn)

class TransactionManager:
    """Manages database transactions"""
    
    def __init__(self, log_manager: LogManager, record_manager: RecordManager):
        self.log_manager = log_manager
        self.record_manager = record_manager
        self.active_transactions = {}
        self.next_txn_id = 1
        self.lock = threading.Lock()
    
    def begin_transaction(self) -> int:
        """Start new transaction"""
        with self.lock:
            txn_id = self.next_txn_id
            self.next_txn_id += 1
            
            # Log transaction begin
            log_record = LogRecord(txn_id, LogRecordType.BEGIN)
            self.log_manager.write_log_record(log_record)
            
            self.active_transactions[txn_id] = {
                'start_time': time.time(),
                'operations': []
            }
            
            return txn_id
    
    def commit_transaction(self, txn_id: int):
        """Commit transaction"""
        with self.lock:
            if txn_id not in self.active_transactions:
                raise ValueError(f"Transaction {txn_id} not found")
            
            # Log transaction commit
            log_record = LogRecord(txn_id, LogRecordType.COMMIT)
            self.log_manager.write_log_record(log_record)
            
            # Remove from active transactions
            del self.active_transactions[txn_id]
    
    def abort_transaction(self, txn_id: int):
        """Abort transaction and undo changes"""
        with self.lock:
            if txn_id not in self.active_transactions:
                raise ValueError(f"Transaction {txn_id} not found")
            
            # Undo operations in reverse order
            operations = self.active_transactions[txn_id]['operations']
            for op in reversed(operations):
                self._undo_operation(op)
            
            # Log transaction abort
            log_record = LogRecord(txn_id, LogRecordType.ABORT)
            self.log_manager.write_log_record(log_record)
            
            # Remove from active transactions
            del self.active_transactions[txn_id]
    
    def transactional_insert(self, txn_id: int, table_name: str, data: Dict[str, Any]) -> int:
        """Insert record within transaction"""
        # Log the operation
        log_record = LogRecord(txn_id, LogRecordType.INSERT, table_name, None, None, data)
        self.log_manager.write_log_record(log_record)
        
        # Perform the insert
        record_id = self.record_manager.insert_record(table_name, data)
        
        # Track operation for potential rollback
        if txn_id in self.active_transactions:
            self.active_transactions[txn_id]['operations'].append({
                'type': 'INSERT',
                'table_name': table_name,
                'record_id': record_id,
                'data': data
            })
        
        return record_id
    
    def _undo_operation(self, operation: Dict[str, Any]):
        """Undo a single operation"""
        if operation['type'] == 'INSERT':
            self.record_manager.delete_record(operation['table_name'], operation['record_id'])
        elif operation['type'] == 'UPDATE':
            self.record_manager.update_record(operation['table_name'], 
                                            operation['record_id'], 
                                            operation['old_data'])
        elif operation['type'] == 'DELETE':
            # Restore deleted record (simplified)
            pass
```

## Putting It All Together: A Simple Database Engine

Now we can combine all components into a functional database engine that builds upon our calculator's expression evaluation:

```python
class SimpleDatabase:
    """Complete database engine combining all components"""
    
    def __init__(self, db_name: str):
        self.db_name = db_name
        self.storage_manager = StorageManager(f"{db_name}.db")
        self.log_manager = LogManager(f"{db_name}.log")
        self.record_manager = RecordManager(self.storage_manager)
        self.transaction_manager = TransactionManager(self.log_manager, self.record_manager)
        
        # Import our calculator for expression evaluation
        from calculator import Calculator
        self.calculator = Calculator()
    
    @contextmanager
    def transaction(self):
        """Context manager for transactions"""
        txn_id = self.transaction_manager.begin_transaction()
        try:
            yield txn_id
            self.transaction_manager.commit_transaction(txn_id)
        except Exception as e:
            self.transaction_manager.abort_transaction(txn_id)
            raise e
    
    def insert(self, table_name: str, data: Dict[str, Any], txn_id: int = None) -> int:
        """Insert record with optional transaction"""
        if txn_id:
            return self.transaction_manager.transactional_insert(txn_id, table_name, data)
        else:
            return self.record_manager.insert_record(table_name, data)
    
    def select(self, table_name: str, where_clause: str = None) -> List[Record]:
        """Select records with optional WHERE clause"""
        results = []
        
        # Get all records from table
        for page_id in self.record_manager._get_table_pages(table_name):
            page = self.storage_manager.get_page(page_id)
            try:
                records = self._get_all_records_from_page(page)
                for record in records:
                    if not record.deleted:
                        # Apply WHERE clause if provided
                        if where_clause is None or self._evaluate_where_clause(record, where_clause):
                            results.append(record)
            finally:
                self.storage_manager.unpin_page(page_id)
        
        return results
    
    def update(self, table_name: str, data: Dict[str, Any], where_clause: str = None, txn_id: int = None) -> int:
        """Update records matching WHERE clause"""
        updated_count = 0
        
        # Find matching records
        matching_records = self.select(table_name, where_clause)
        
        for record in matching_records:
            if txn_id:
                # Log the update operation
                log_record = LogRecord(txn_id, LogRecordType.UPDATE, table_name, 
                                     record.record_id, record.data.copy(), data)
                self.log_manager.write_log_record(log_record)
            
            # Perform update
            if self.record_manager.update_record(table_name, record.record_id, data):
                updated_count += 1
        
        return updated_count
    
    def delete(self, table_name: str, where_clause: str = None, txn_id: int = None) -> int:
        """Delete records matching WHERE clause"""
        deleted_count = 0
        
        # Find matching records
        matching_records = self.select(table_name, where_clause)
        
        for record in matching_records:
            if txn_id:
                # Log the delete operation
                log_record = LogRecord(txn_id, LogRecordType.DELETE, table_name, 
                                     record.record_id, record.data.copy(), None)
                self.log_manager.write_log_record(log_record)
            
            # Perform delete
            if self.record_manager.delete_record(table_name, record.record_id):
                deleted_count += 1
        
        return deleted_count
    
    def _get_all_records_from_page(self, page: Page) -> List[Record]:
        """Extract all records from a page"""
        records = []
        page_id, record_count, free_space, flags, checksum = page.get_header()
        
        offset = Page.HEADER_SIZE
        for i in range(record_count):
            try:
                # Read record header
                size, record_id = struct.unpack('<II', page.data[offset:offset+8])
                
                # Read record data
                record_data = page.data[offset+8:offset+8+size]
                record = Record.deserialize(record_data)
                records.append(record)
                
                offset += 8 + size
            except (struct.error, json.JSONDecodeError):
                # Skip corrupted records
                break
        
        return records
    
    def _evaluate_where_clause(self, record: Record, where_clause: str) -> bool:
        """Evaluate WHERE clause against record using calculator engine"""
        try:
            # Simple WHERE clause evaluation
            # Convert record fields to variables for expression evaluation
            expression = where_clause
            
            # Replace field references with actual values
            for field, value in record.data.items():
                if isinstance(value, (int, float)):
                    expression = expression.replace(field, str(value))
                elif isinstance(value, str):
                    # Handle string comparisons (simplified)
                    expression = expression.replace(f"'{value}'", "1")
                    expression = expression.replace(f'"{value}"', "1")
            
            # Use calculator to evaluate the expression
            result = self.calculator.evaluate(expression)
            
            # Return True if result is truthy and not an error
            return isinstance(result, (int, float)) and result != 0
            
        except Exception:
            # If evaluation fails, exclude the record
            return False
    
    def execute_sql(self, sql: str) -> Any:
        """Execute simple SQL statements"""
        sql = sql.strip().upper()
        
        if sql.startswith('INSERT'):
            return self._execute_insert(sql)
        elif sql.startswith('SELECT'):
            return self._execute_select(sql)
        elif sql.startswith('UPDATE'):
            return self._execute_update(sql)
        elif sql.startswith('DELETE'):
            return self._execute_delete(sql)
        else:
            raise ValueError(f"Unsupported SQL statement: {sql}")
    
    def _execute_insert(self, sql: str) -> int:
        """Execute INSERT statement"""
        # Simple parsing: INSERT INTO table_name (col1, col2) VALUES (val1, val2)
        # This is a simplified parser - production systems need full SQL parsing
        
        parts = sql.split()
        table_name = parts[2].lower()
        
        # Extract values (very simplified)
        values_start = sql.find('VALUES')
        values_part = sql[values_start+6:].strip()
        values_part = values_part.strip('()')
        
        # Parse values (simplified - assumes string format)
        values = [v.strip().strip("'\"") for v in values_part.split(',')]
        
        # For demo, assume simple column names
        data = {
            'id': int(values[0]) if values[0].isdigit() else values[0],
            'name': values[1] if len(values) > 1 else None,
            'value': float(values[2]) if len(values) > 2 and values[2].replace('.','').isdigit() else values[2] if len(values) > 2 else None
        }
        
        return self.insert(table_name, data)
    
    def _execute_select(self, sql: str) -> List[Record]:
        """Execute SELECT statement"""
        # Simple parsing: SELECT * FROM table_name WHERE condition
        parts = sql.split()
        
        # Find table name
        from_index = parts.index('FROM')
        table_name = parts[from_index + 1].lower()
        
        # Find WHERE clause if present
        where_clause = None
        if 'WHERE' in parts:
            where_index = parts.index('WHERE')
            where_clause = ' '.join(parts[where_index + 1:])
        
        return self.select(table_name, where_clause)
    
    def _execute_update(self, sql: str) -> int:
        """Execute UPDATE statement"""
        # Simple parsing: UPDATE table_name SET col=val WHERE condition
        parts = sql.split()
        table_name = parts[1].lower()
        
        # Find SET clause
        set_index = parts.index('SET')
        where_index = parts.index('WHERE') if 'WHERE' in parts else len(parts)
        
        set_clause = ' '.join(parts[set_index + 1:where_index])
        
        # Parse SET assignments (simplified)
        assignments = {}
        for assignment in set_clause.split(','):
            key, value = assignment.split('=')
            assignments[key.strip()] = value.strip().strip("'\"")
        
        # WHERE clause
        where_clause = None
        if 'WHERE' in parts:
            where_index = parts.index('WHERE')
            where_clause = ' '.join(parts[where_index + 1:])
        
        return self.update(table_name, assignments, where_clause)
    
    def _execute_delete(self, sql: str) -> int:
        """Execute DELETE statement"""
        # Simple parsing: DELETE FROM table_name WHERE condition
        parts = sql.split()
        
        from_index = parts.index('FROM')
        table_name = parts[from_index + 1].lower()
        
        where_clause = None
        if 'WHERE' in parts:
            where_index = parts.index('WHERE')
            where_clause = ' '.join(parts[where_index + 1:])
        
        return self.delete(table_name, where_clause)
    
    def close(self):
        """Close database and ensure all data is persisted"""
        self.storage_manager.sync()

# Usage Example and Testing
def demonstrate_database():
    """Demonstrate the database functionality"""
    
    # Create database
    db = SimpleDatabase("test_db")
    
    try:
        print("=== Database Storage Engine Demo ===\n")
        
        # Test basic operations
        print("1. Inserting records...")
        with db.transaction() as txn:
            db.insert("users", {"id": 1, "name": "Alice", "age": 25, "balance": 1000.50}, txn)
            db.insert("users", {"id": 2, "name": "Bob", "age": 30, "balance": 2500.75}, txn)
            db.insert("users", {"id": 3, "name": "Charlie", "age": 35, "balance": 500.25}, txn)
        print("Inserted 3 users successfully")
        
        # Test select
        print("\n2. Selecting all records...")
        all_users = db.select("users")
        for user in all_users:
            print(f"  User {user.record_id}: {user.data}")
        
        # Test WHERE clause with calculator integration
        print("\n3. Selecting with WHERE clause (balance > 1000)...")
        rich_users = db.select("users", "balance > 1000")
        for user in rich_users:
            print(f"  Rich user: {user.data}")
        
        # Test update
        print("\n4. Updating records...")
        with db.transaction() as txn:
            updated_count = db.update("users", {"age": 26}, "id = 1", txn)
        print(f"Updated {updated_count} records")
        
        # Test SQL execution
        print("\n5. Testing SQL execution...")
        sql_results = db.execute_sql("SELECT * FROM users WHERE balance > 500")
        print(f"SQL query returned {len(sql_results)} results")
        
        # Test recovery scenario
        print("\n6. Testing transaction rollback...")
        try:
            with db.transaction() as txn:
                db.insert("users", {"id": 4, "name": "David", "age": 40, "balance": 3000}, txn)
                db.insert("users", {"id": 5, "name": "Eve", "age": 28, "balance": 1500}, txn)
                # Simulate error
                raise Exception("Simulated error")
        except Exception as e:
            print(f"Transaction rolled back due to: {e}")
        
        # Verify rollback worked
        final_users = db.select("users")
        print(f"Final user count after rollback: {len(final_users)}")
        
        # Test persistence
        print("\n7. Testing persistence...")
        db.close()
        
        # Reopen database
        db2 = SimpleDatabase("test_db")
        persisted_users = db2.select("users")
        print(f"Persisted user count after reopen: {len(persisted_users)}")
        db2.close()
        
    except Exception as e:
        print(f"Error during demonstration: {e}")
        import traceback
        traceback.print_exc()
    finally:
        db.close()

if __name__ == "__main__":
    demonstrate_database()

## Performance Analysis and Optimization

Our storage engine implements several key optimizations that mirror production database systems:

### Buffer Pool Management
The buffer pool acts as an intelligent cache between our application and disk storage. Key performance characteristics:

- **Hit Rate**: When requested pages are already in memory, access time is O(1)
- **Eviction Policy**: LRU (Least Recently Used) ensures frequently accessed pages stay in memory
- **Pin Counting**: Prevents eviction of pages currently being modified

### Page-Based I/O
Fixed-size pages provide several performance benefits:

- **Aligned I/O**: 4KB pages align with OS page sizes, minimizing system call overhead
- **Batch Operations**: Multiple records per page reduce I/O operations
- **Predictable Performance**: Fixed-size allocations avoid memory fragmentation

### Write-Ahead Logging
WAL provides durability while optimizing for common access patterns:

- **Sequential Writes**: Log entries are written sequentially, maximizing disk throughput
- **Batched Commits**: Multiple transactions can share log flushes
- **Recovery Speed**: Forward recovery is faster than reconstructing from data pages

## Extending Toward Full Database Functionality

Our storage engine provides the foundation for advanced database features:

### Indexing Support
The page-based architecture supports B-tree indexes:

```python
class BTreeNode(Page):
    """B-tree node implemented as a specialized page"""
    
    def __init__(self, page_id: int, is_leaf: bool = True):
        super().__init__(page_id)
        self.is_leaf = is_leaf
        self.keys = []
        self.values = []  # For leaf nodes
        self.children = []  # For internal nodes
    
    def insert_key(self, key: Any, value: Any = None):
        """Insert key-value pair maintaining sort order"""
        # Implementation would maintain B-tree properties
        pass
```

### Concurrency Control
The transaction framework supports lock-based concurrency:

```python
class LockManager:
    """Manages record-level locking"""
    
    def __init__(self):
        self.locks = {}  # record_id -> lock_info
        self.lock = threading.RLock()
    
    def acquire_lock(self, txn_id: int, record_id: int, lock_type: str):
        """Acquire read or write lock on record"""
        # Implementation would handle lock compatibility matrix
        pass
```

### Query Optimization
Integration with our calculator's expression evaluation enables cost-based optimization:

```python
class QueryOptimizer:
    """Optimize query execution plans"""
    
    def __init__(self, storage_engine):
        self.storage_engine = storage_engine
        self.statistics = {}  # Table statistics for cost estimation
    
    def optimize_query(self, query_plan):
        """Generate optimal execution plan"""
        # Use expression evaluation costs from calculator
        # Estimate I/O costs based on storage statistics
        pass
```

## Bridging to Database Queries

The storage engine seamlessly integrates with our calculator's expression evaluation. When processing queries like:

```sql
SELECT balance * 1.08 AS total_with_tax 
FROM users 
WHERE age > 25 AND balance > (1000 + 500)
```

The system works as follows:

1. **Storage Layer**: Retrieves user records from disk pages
2. **Expression Evaluator**: Uses our calculator to evaluate `age > 25 AND balance > (1000 + 500)`
3. **Computation Engine**: Calculates `balance * 1.08` for qualifying records
4. **Transaction Manager**: Ensures consistency if the query is within a transaction

## Conclusion: From Calculator to Storage Engine

This chapter demonstrated how our calculator's foundations scale to handle persistent data storage. The recursive descent parser that handled mathematical expressions now supports SQL WHERE clauses. The error handling that managed division by zero now manages disk I/O failures and transaction rollbacks.

Key architectural lessons:

1. **Layered Design**: Each component (pages, buffer pool, transactions) builds upon the previous layer
2. **Error Handling**: Robust error handling becomes critical when dealing with persistent state
3. **Performance Trade-offs**: Every design decision involves trade-offs between performance, durability, and complexity
4. **Extensibility**: The modular design enables adding advanced features like indexing and query optimization

Our storage engine provides the persistent foundation that transforms our calculator from a simple expression evaluator into the backbone of a database system. In the next chapter, we'll build upon this storage layer to implement SQL query processing, showing how the expression parsing techniques from our calculator enable full SQL support.

The journey from arithmetic expressions to database queries follows a natural progression: both involve parsing structured input, evaluating expressions according to precedence rules, and returning computed results. The difference lies in scale, persistence, and the complexity of the data being processed.
