```python
import os
import re
import unicodedata
import pandas as pd

# === 路径（与你的风格一致）===
script_dir = os.path.dirname(os.path.realpath(__file__))
file_a = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))
file_b = os.path.abspath(os.path.join(script_dir, "golden.csv"))  # 改成你的B文件名
output_file = os.path.abspath(os.path.join(script_dir, "svocid_diff.csv"))

# === 小工具：规范化列名 ===
def norm_col(s: str) -> str:
    s = unicodedata.normalize("NFKC", str(s)).replace("\ufeff", "")
    s = s.strip().lower()
    # 去掉空格和大部分符号，但保留冒号和下划线，方便匹配 custom:svoc_id
    s = re.sub(r"[^\w:]", "", s)     # \w 保留 [a-z0-9_]
    # 把 svocid / svoc_id 统一到 svoc_id
    s = s.replace("svocid", "svoc_id")
    return s

def build_colmap(cols):
    """把原始列名映射到规范化名 -> 原名"""
    m = {}
    for c in cols:
        m[norm_col(c)] = c
    return m

def pick(colmap, candidates):
    """在候选规范名里找第一个存在的，返回原始列名；找不到抛错并打印可用列"""
    for cand in candidates:
        key = norm_col(cand)
        if key in colmap:
            return colmap[key]
    raise KeyError(f"找不到列，候选={candidates}；可用列(规范化后)={list(colmap.keys())}")

# === 读取 ===
df_a = pd.read_csv(file_a, dtype=str)
df_b = pd.read_csv(file_b, dtype=str)

# === 构建列名映射（原名 <-> 规范名）===
map_a = build_colmap(df_a.columns)
map_b = build_colmap(df_b.columns)

# A 文件期望的列：NRIC 与 SVOCID（可能是 custom:nric / custom:svoc_id）
col_a_nric = pick(map_a, ["custom:nric", "nric"])
col_a_svoc = pick(map_a, ["custom:svoc_id", "custom:svocid", "svoc_id", "svocid"])

# B 文件期望的列：NRIC 与 SVOCID（可能没有 custom: 前缀）
col_b_nric = pick(map_b, ["nric", "custom:nric"])
col_b_svoc = pick(map_b, ["svoc_id", "svocid", "custom:svoc_id", "custom:svocid"])

# 只保留需要的列并去空
df_a_sub = df_a[[col_a_nric, col_a_svoc]].dropna()
df_b_sub = df_b[[col_b_nric, col_b_svoc]].dropna()

# 统一列名，方便合并
df_a_sub = df_a_sub.rename(columns={col_a_nric: "nric", col_a_svoc: "svocid_a"})
df_b_sub = df_b_sub.rename(columns={col_b_nric: "nric", col_b_svoc: "svocid_b"})

# 合并并找差异
merged = pd.merge(df_a_sub, df_b_sub, on="nric", how="inner")
diff = merged[merged["svocid_a"] != merged["svocid_b"]][["nric", "svocid_a"]]

# 保存
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存结果到: {output_file}")
