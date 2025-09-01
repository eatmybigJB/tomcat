```python
import os
import re
import unicodedata
import pandas as pd

# === 路径（与你的风格一致）===
script_dir = os.path.dirname(os.path.realpath(__file__))
file_a = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))  # A 文件
file_b = os.path.abspath(os.path.join(script_dir, "golden.csv"))                     # B 文件（改成你的文件名）
output_file = os.path.abspath(os.path.join(script_dir, "svocid_diff_step2.csv"))    # 输出文件名按需改

# === 小工具：规范化列名 ===
def norm_col(s: str) -> str:
    s = unicodedata.normalize("NFKC", str(s)).replace("\ufeff", "")
    s = s.strip().lower()
    # 去掉大部分符号，但保留 冒号(:) 和 下划线(_) 以便匹配 custom:svoc_id
    s = re.sub(r"[^\w:]", "", s)      # \w 等价于 [a-zA-Z0-9_]
    # 把 svocid / svoc_id 统一成 svoc_id
    s = s.replace("svocid", "svoc_id")
    return s

def build_colmap(cols):
    """把原始列名映射到 规范化名 -> 原名"""
    m = {}
    for c in cols:
        m[norm_col(c)] = c
    return m

def pick(colmap, candidates):
    """
    在候选规范名里找第一个存在的，返回“原始列名”；
    找不到会抛错并打印可用列（规范化后）。
    """
    for cand in candidates:
        key = norm_col(cand)
        if key in colmap:
            return colmap[key]
    raise KeyError(f"找不到列，候选={candidates}；可用列(规范化后)={list(colmap.keys())}")

# === 读取 CSV（全部按字符串读，避免类型问题）===
df_a = pd.read_csv(file_a, dtype=str)
df_b = pd.read_csv(file_b, dtype=str)

# === 构建列名映射（原名 <-> 规范名）===
map_a = build_colmap(df_a.columns)
map_b = build_colmap(df_b.columns)

# === A 文件列选择：NRIC、SVOCID、preferred_username、phone_number ===
col_a_nric = pick(map_a, ["custom:nric", "nric"])
col_a_svoc = pick(map_a, ["custom:svoc_id", "custom:svocid", "svoc_id", "svocid"])

# 这两列有可能不存在；如果不存在就跳过
col_a_user = map_a.get(norm_col("preferred_username"))
col_a_phone = map_a.get(norm_col("phone_number"))

use_cols_a = [col_a_nric, col_a_svoc]
if col_a_user:  # 只有存在才取
    use_cols_a.append(col_a_user)
if col_a_phone:
    use_cols_a.append(col_a_phone)

df_a_sub = df_a[use_cols_a].copy()
df_a_sub = df_a_sub.rename(columns={
    col_a_nric: "nric",
    col_a_svoc: "svocid_a",
    **({col_a_user: "preferred_username"} if col_a_user else {}),
    **({col_a_phone: "phone_number"} if col_a_phone else {}),
})

# === B 文件列选择：NRIC、SVOCID（可能有 custom: 前缀）===
col_b_nric = pick(map_b, ["nric", "custom:nric"])
col_b_svoc = pick(map_b, ["svoc_id", "svocid", "custom:svoc_id", "custom:svocid"])

df_b_sub = df_b[[col_b_nric, col_b_svoc]].copy()
df_b_sub = df_b_sub.rename(columns={col_b_nric: "nric", col_b_svoc: "svocid_b"})

# 仅对 nric 做非空过滤，避免因为某一边 svocid 空导致过早丢数据
df_a_sub = df_a_sub.dropna(subset=["nric"])
df_b_sub = df_b_sub.dropna(subset=["nric"])

# === 合并（按 nric）===
merged = pd.merge(df_a_sub, df_b_sub, on="nric", how="inner")

# === 取出 svocid 不一致的记录 ===
keep_cols = ["nric", "svocid_a", "svocid_b"]
if "preferred_username" in merged.columns:
    keep_cols.insert(1, "preferred_username")
if "phone_number" in merged.columns:
    keep_cols.insert(2 if "preferred_username" in merged.columns else 1, "phone_number")

diff = merged.loc[merged["svocid_a"] != merged["svocid_b"], keep_cols]

# === 去重（整行去重；若只想按 nric 去重，可改为 diff = diff.drop_duplicates(subset=["nric"])）===
diff = diff.drop_duplicates()

# === 保存 ===
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存结果到: {output_file}")
