```python
import csv
from itertools import islice
from typing import Iterable, Dict, Any, List, Optional, Tuple
from concurrent.futures import ThreadPoolExecutor, wait, FIRST_COMPLETED

def write_users_to_two_csvs_mt(
    users_iter: Iterable[Dict[str, Any]],
    tmp_csv_path: str,
    tmp_csv_path_full: str,
    attributes_full: List[str],
    attributes_limit: List[str],
    *,
    limit: Optional[int] = None,
    batch_size: int = 4000,
    max_workers: int = 8,
) -> None:
    """
    单次遍历 users_iter，同时生成两份 CSV：
      - tmp_csv_path:       只写 attributes_limit
      - tmp_csv_path_full:  写 attributes_full
    使用线程池并发处理批次（CPU/JSON投影），文件写入在主线程完成（避免锁争用与乱序）。
    """

    def process_batch(batch: List[Dict[str, Any]]) -> Tuple[List[Dict[str, Any]], List[Dict[str, Any]]]:
        # 在工作线程里完成字段投影与缺省填充
        rows_small = [{k: u.get(k, "") for k in attributes_limit} for u in batch]
        rows_full  = [{k: u.get(k, "") for k in attributes_full}  for u in batch]
        return rows_small, rows_full

    with open(tmp_csv_path, 'w', newline='', encoding='utf-8') as f_small, \
         open(tmp_csv_path_full, 'w', newline='', encoding='utf-8') as f_full, \
         ThreadPoolExecutor(max_workers=max_workers) as executor:

        w_small = csv.DictWriter(f_small, fieldnames=attributes_limit, extrasaction='ignore')
        w_full  = csv.DictWriter(f_full,  fieldnames=attributes_full,  extrasaction='ignore')
        w_small.writeheader()
        w_full.writeheader()

        remaining = limit
        futures = []

        while True:
            n_take = batch_size if remaining is None else max(0, min(batch_size, remaining))
            if n_take == 0 and remaining is not None:
                break

            batch = list(islice(users_iter, n_take if n_take else batch_size))
            if not batch:
                break

            futures.append(executor.submit(process_batch, batch))

            # 控制并发窗口，避免占用过多内存
            if len(futures) >= max_workers:
                done, futures = wait(futures, return_when=FIRST_COMPLETED), [f for f in futures if not f.done()]
                for d in done.done:
                    rows_small, rows_full = d.result()
                    w_small.writerows(rows_small)
                    w_full.writerows(rows_full)

            if remaining is not None:
                remaining -= len(batch)
                if remaining <= 0:
                    break

        # 把剩余批次写完
        for f in futures:
            rows_small, rows_full = f.result()
            w_small.writerows(rows_small)
            w_full.writerows(rows_full)

write_users_to_two_csvs_mt(
    users_gen,
    tmp_csv_path=tmp_csv_path,
    tmp_csv_path_full=tmp_csv_path_full,
    attributes_full=attributes_full,
    attributes_limit=attributes_limit,
    batch_size=4000,
    max_workers=8,
)
