# LSMQueryDrivenCompaction
Analyzing the performance implications of range query-driven compactions in LSMs

## Problem Statement
The state-of-the-art LSM-based data stores perform range queries by retrieving multiple files from various levels and filtering out invalid keys. Once the range query is complete, the work done via the sort-merge operation for the query is discarded. The results are neither cached nor flushed back to the LSM tree except by returning it to the application. This approach can give rise to two problems: (1)Redundant Work: when the same range query or an overlapping one is executed again, and (2) Increased Write Amplification: when a compaction is triggered for the files containing keys within the range of a previous range query. During these processes, the system reads the invalid keys and subsequently either drops them or replaces them with new values. As a result, the same data bytes are read and written during both the range query and compaction processes, leading to higher read, write, and space amplification.

## Solution "Query-Driven Compaction"

The flow of range query in query-driven compaction is shown in Figure. The initial setup of level iterators goes the same as in the state-of-the-art LSM range query. Once all the iterators are initialized, it performs a seek operation on iterators using the range-query `start_key` for each level. The seek operation in query-driven compaction performs an extra operation to create a partial file flush job and add it to the `write_queue_`. The partial file flush will only happen if the file does not completely overlap with the range-query `start_key` and `end_key`. We can have three scenarios for the partial file flush, which (as shown in Figure) are as follows.

![Image1](./File%20range%20overlaps.png)

1. **No overlap** In this scenario, the partial file flush will not happen as the file does not overlap. Figure (e)

2. **Smallest or Largest key overlap** (_Partial Flush_) In this, the smallest or largest key of the file overlaps with the range query start or end key. Figure (a) and (b)

3. **Head or Tail overlap** (_Partial Flush_) In this, the partial part (having more than one key) overlaps with the range query. Figure (c) and (d)

4. **The range fits inside file overlap** (_Partial-Partial Flush_) Both the start and end keys of the range query fit completely in the file’s smallest and largest keys. Figure-4 (f)

These partial flush jobs, that are added to the write_queue_ are executed by the 𝑃𝑟𝑖𝑜𝑟𝑖𝑡𝑦::𝐻𝐼𝐺𝐻 background thread, parallel to the range query. It flushes the partial part of the file that does not fall in the range query to the same level in the LSM tree. The files that are completely overlapping with the range query `start_key` and `end_key` will be deleted from the lower levels and the valid keys will be added to the higher levels in new files. We can have one scenario for the file deletion from the lower levels which is as follows. 

1. **Complete overlap i.e. start_key <= smallest key and end_key >= largest key** (_Just Delete_) All files in the lower level, as well as in the higher level, that are completely overlapping with the range query will be deleted by the query-driven compaction. Figure(g)

The query-driven compaction also initiates an in-memory buffer of the default size configured in db_options to keep copies of the valid keys and flush them back to the LSM when it is full. The main thread, that is executing the range query will keep on performing the sort-merge operation and return the valid keys to the application. Whenever it returns a valid key to the application, the query-driven compaction makes a copy and stores it in a memtable. Once the memtable is full, it creates a new flush job and adds it to the `write_queue_`. The query-driven compaction also adds write stalls for each range query to flush all the jobs initiated during this operation. This happens by stopping the background work during the start of the range query and waiting for all the flush jobs to execute successfully at the end of the range query. Whenever query-driven compaction is triggered during the range query, it will remove all the logically invalid keys and tombstones from the LSM tree that fall in that range, which also results in reduced space amplification. This way whenever background compactions are triggered after successful query-driven compaction, it will increase the chances of trivial moves of files between levels as per the minimum overlapping strategy. It will also reduce the write amplification for any compaction that has overlapping keys with the previous query-driven compactions. Keys from Level-0 are not pushed to the last level and there would be no partial flush for the same to keep the hot data in lower levels.

![Image1](./RQ-driven%20numeric%20key%20sorting.png)
