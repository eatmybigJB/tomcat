```python
import os
import re
import unicodedata
import pandas as pd

# === 路径（与你的风格一致）===
script_dir = os.path.dirname(os.path.realpath(__file__))
file_a = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))  # A文件
file_b = os.path.abspath(os.path.join(script_dir, "golden.csv"))                     # B文件(按你的文件名改)
output_file = os.path.abspath(os.path.join(script_dir, "svocid_diff_with_user_phone.csv"))

# === 列名规范化工具 ===
def norm_col(s: str) -> str:
    s = unicodedata.normalize("NFKC", str(s)).replace("\ufeff", "")
    s = s.strip().lower()
    # 保留字母数字下划线与冒号；去掉其它符号（便于匹配 custom:svoc_id）
    s = re.sub(r"[^\w:]", "", s)
    # 统一 svocid / svoc_id
    s = s.replace("svocid", "svoc_id")
    return s

def build_colmap(cols):
    """原名列表 -> {规范名: 原名}"""
    m = {}
    for c in cols:
        m[norm_col(c)] = c
    return m

def pick(colmap, candidates):
    """从候选规范名中找到第一列并返回原始列名，否则报错（打印可用列帮助定位）"""
    for cand in candidates:
        key = norm_col(cand)
        if key in colmap:
            return colmap[key]
    raise KeyError(f"找不到列，候选={candidates}；可用列(规范化后)={list(colmap.keys())}")

# === 读取 CSV ===
df_a = pd.read_csv(file_a, dtype=str)
df_b = pd.read_csv(file_b, dtype=str)

# === 构建映射 ===
map_a = build_colmap(df_a.columns)
map_b = build_colmap(df_b.columns)

# === A 文件列 ===
# nric / svocid
col_a_nric = pick(map_a, ["custom:nric", "nric"])
col_a_svoc = pick(map_a, ["custom:svoc_id", "custom:svocid", "svoc_id", "svocid"])
# 你截图中的这两列
col_a_user = pick(map_a, ["hscb_cognito_preferred_username", "preferred_username"])
col_a_phone = pick(map_a, ["hscb_cognito_phone", "phone_number"])

df_a_sub = df_a[[col_a_nric, col_a_svoc, col_a_user, col_a_phone]].copy()
df_a_sub = df_a_sub.rename(columns={
    col_a_nric: "nric",
    col_a_svoc: "svocid_a",
    col_a_user: "hscb_cognito_preferred_username",
    col_a_phone: "hscb_cognito_phone",
})
df_a_sub = df_a_sub.dropna(subset=["nric"])

# === B 文件列（nric、svocid，可能带 custom: 前缀）===
col_b_nric = pick(map_b, ["nric", "custom:nric"])
col_b_svoc = pick(map_b, ["svoc_id", "svocid", "custom:svoc_id", "custom:svocid"])

df_b_sub = df_b[[col_b_nric, col_b_svoc]].copy()
df_b_sub = df_b_sub.rename(columns={col_b_nric: "nric", col_b_svoc: "svocid_b"})
df_b_sub = df_b_sub.dropna(subset=["nric"])

# === 合并并取差异 ===
merged = pd.merge(df_a_sub, df_b_sub, on="nric", how="inner")

# 仅保留 svocid 不一致的记录
keep_cols = ["nric", "hscb_cognito_preferred_username", "hscb_cognito_phone", "svocid_a", "svocid_b"]
diff = merged.loc[merged["svocid_a"] != merged["svocid_b"], keep_cols]

# 去重（整行）
diff = diff.drop_duplicates()

# === 保存 ===
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存结果到: {output_file}")

