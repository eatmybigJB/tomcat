```python
import os
import re
import unicodedata
import pandas as pd

# === 路径（与你的风格一致） ===
script_dir  = os.path.dirname(os.path.realpath(__file__))
file_a      = os.path.abspath(os.path.join(script_dir, "email_same.csv"))            # A 文件
file_b      = os.path.abspath(os.path.join(script_dir, "svocid_diff_step1.csv"))     # B 文件
output_file = os.path.abspath(os.path.join(script_dir, "svocid_diff_step2.csv"))     # 输出文件
lookup_file = os.path.abspath(os.path.join(script_dir, "cep_svocid_email_phone.csv"))# 追加映射文件

# === 列名规范化工具 ===
def norm_col(s: str) -> str:
    s = unicodedata.normalize("NFKC", str(s)).replace("\ufeff", "")
    s = s.strip().lower()
    # 保留字母/数字/下划线；去掉其它符号（便于匹配）
    s = re.sub(r"[^\w]+", "_", s)
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
    """
    从候选规范名中找到第一个并返回原始列名；否则报错（打印可用帮助定位）
    """
    for cand in candidates:
        key = norm_col(cand)
        if key in colmap:
            return colmap[key]
    raise KeyError(f"找不到列, 传入={candidates}; 可用列(规范化后)={list(colmap.keys())}")

# === 读取 CSV ===
df_a = pd.read_csv(file_a, dtype=str)
df_b = pd.read_csv(file_b, dtype=str)

# === 构建映射 ===
map_a = build_colmap(df_a.columns)
map_b = build_colmap(df_b.columns)

# === A 文件列（nric / svocid） ===
col_a_nric   = pick(map_a, ["nric"])
col_a_svoc   = pick(map_a, ["svocid"])  # A 侧通常是 hscb_cognito_svocid
col_a_user   = pick(map_a, ["hscb_cognito_username"])
col_a_phone  = pick(map_a, ["hscb_cognito_phone"])
col_a_status = pick(map_a, ["hscb_cognito_user_status"])

df_a_sub = df_a[[col_a_nric, col_a_svoc, col_a_user, col_a_phone, col_a_status]].copy()
df_a_sub = df_a_sub.rename(columns={
    col_a_nric:   "nric",
    col_a_svoc:   "hscb_cognito_svocid",
    col_a_user:   "hscb_cognito_username",
    col_a_phone:  "hscb_cognito_phone",
    col_a_status: "hscb_cognito_user_status",
})
df_a_sub = df_a_sub.dropna(subset=["nric"])

# === B 文件列（nric, svocid，可能带 custom: 前缀） ===
col_b_nric = pick(map_b, ["nric"])
col_b_svoc = pick(map_b, ["svoc_id", "svocid"])

df_b_sub = df_b[[col_b_nric, col_b_svoc]].copy()
df_b_sub = df_b_sub.rename(columns={col_b_nric: "nric", col_b_svoc: "cep_svocid"})
df_b_sub = df_b_sub.dropna(subset=["nric"])

# === 合并并找差 ===
merged = pd.merge(df_a_sub, df_b_sub, on="nric", how="inner")

keep_cols = ["nric",
             "hscb_cognito_username",
             "hscb_cognito_phone",
             "hscb_cognito_svocid",
             "cep_svocid"]

diff = merged.loc[merged["hscb_cognito_svocid"] != merged["cep_svocid"], keep_cols]

# 去重（保留意义相同的行只留一条）
diff = diff.drop_duplicates()

# === 在写出前：按 cep_svocid 追加 cep_email / cep_phone ===
def _clean_series(s: pd.Series) -> pd.Series:
    # 仅做轻度清洗：去 BOM/首尾空白；绝不移除 '+' 号
    return (
        s.astype(str)
         .str.replace("\ufeff", "", regex=False)
         .str.strip()
    )

try:
    df_c = pd.read_csv(lookup_file, dtype=str)
    map_c = build_colmap(df_c.columns)

    col_c_svoc  = pick(map_c, ["cep_svocid", "svoc_id", "svocid"])
    col_c_email = pick(map_c, ["cep_email", "email"])
    col_c_phone = pick(map_c, ["cep_phone", "phone"])

    df_lookup = df_c[[col_c_svoc, col_c_email, col_c_phone]].copy()
    df_lookup = df_lookup.rename(columns={
        col_c_svoc:  "cep_svocid",
        col_c_email: "cep_email",
        col_c_phone: "cep_phone",
    })

    for cc in ["cep_svocid", "cep_email", "cep_phone"]:
        df_lookup[cc] = _clean_series(df_lookup[cc])

    # 丢弃空 key；按 cep_svocid 去重（取最后一条以覆盖旧值）
    df_lookup = (
        df_lookup
        .dropna(subset=["cep_svocid"])
        .drop_duplicates(subset=["cep_svocid"], keep="last")
    )

    # 左连接把两列补到 diff（新增列会自动追加在最后）
    diff = diff.merge(df_lookup, on="cep_svocid", how="left")

except FileNotFoundError:
    print(f"[WARN] 未找到联系人映射文件：{lookup_file}，将不追加 cep_email/cep_phone。")
except KeyError as e:
    print(f"[WARN] 解析联系人映射文件列失败：{e}，将不追加 cep_email/cep_phone。")

# === 写出 ===
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存到结果表：{output_file}")







询问 ChatGPT
