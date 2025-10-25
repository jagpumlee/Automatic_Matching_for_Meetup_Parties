import streamlit as st
import pandas as pd
import io
from datetime import datetime
import numpy as np

def detect_gender_split(df):
    # 여성 구분 키워드
    keywords = ["여성참가자", "여성", "여자참가자", "여자", "woman", "female"]
    for i, row in df.iterrows():
        for col in row:
            if isinstance(col, str) and any(k.lower() in col.lower() for k in keywords):
                return i
    return None

def normalize_columns(df):
    mapping = {
        "번호": "번호", "no": "번호", "id": "번호",
        "성명": "이름", "이름": "이름", "name": "이름",
        "1지망": "1지망", "희망1": "1지망", "1순위": "1지망",
        "2지망": "2지망", "희망2": "2지망", "2순위": "2지망",
        "3지망": "3지망", "희망3": "3지망", "3순위": "3지망",
        "4지망": "4지망", "희망4": "4지망", "4순위": "4지망"
    }
    new_cols = []
    for c in df.columns:
        key = str(c).strip().lower()
        found = None
        for k, v in mapping.items():
            if k.lower() == key:
                found = v
                break
        new_cols.append(found if found else c)
    df.columns = new_cols
    return df

def clean_preferences(val):
    if pd.isna(val):
        return None
    s = str(val).strip()
    if s == '' or s == '-' or s == '—':
        return None
    if not s.isdigit():
        return None
    return int(s)

def parse_participants(df):
    df = normalize_columns(df)
    df = df.dropna(how='all')
    df['번호'] = df['번호'].astype(int)
    for col in ['1지망', '2지망', '3지망', '4지망']:
        if col in df.columns:
            df[col] = df[col].apply(clean_preferences)
    return df

def build_match_candidates(men, women):
    candidates = []
    pref_levels = [("1지망", "1지망"), ("1지망", "2지망"), ("2지망", "1지망"),
                   ("1지망", "3지망"), ("3지망", "1지망"), ("2지망", "2지망"),
                   ("2지망", "3지망"), ("3지망", "2지망"), ("3지망", "3지망"),
                   ("1지망", "4지망"), ("4지망", "1지망"), ("2지망", "4지망"),
                   ("4지망", "2지망"), ("3지망", "4지망"), ("4지망", "3지망"),
                   ("4지망", "4지망")]
    for wi, wf in women.iterrows():
        for mi, mf in men.iterrows():
            for rank_w, rank_m in pref_levels:
                if rank_w in women.columns and rank_m in men.columns:
                    if wf[rank_w] == mf['번호'] and mf[rank_m] == wf['번호']:
                        candidates.append({
                            '여성번호': wf['번호'], '여성이름': wf['이름'],
                            '남성번호': mf['번호'], '남성이름': mf['이름'],
                            '우선순위': f"{rank_w[0]}-{rank_m[0]}"
                        })
                        break
    return candidates

def resolve_duplicates(candidates):
    used_m = set()
    used_f = set()
    final = []
    discarded = []
    for c in candidates:
        if c['남성번호'] not in used_m and c['여성번호'] not in used_f:
            final.append(c)
            used_m.add(c['남성번호'])
            used_f.add(c['여성번호'])
        else:
            discarded.append({**c, '사유': '동일 참가자 중 낮은 우선순위'})
    return final, discarded

def print_result_table(final, men_count, women_count, filename, discarded):
    run_id = datetime.now().strftime("%Y-%m-%d %H:%M KST")
    st.write(f"Run ID({run_id}) | 파일명 {filename} | 남(N)={men_count}, 여(N)={women_count}")
    df_out = pd.DataFrame(final)
    if not df_out.empty:
        df_out = df_out[['우선순위', '남성번호', '남성이름', '여성번호', '여성이름']]
        df_out = df_out.sort_values(by='남성번호')
        df_out.index = np.arange(1, len(df_out)+1)
        df_out.index.name = '커플 순번'
        st.dataframe(df_out)
        st.write(f"총 {men_count+women_count}명 중 {len(final)*2}명({len(final)}쌍) 썸매칭 성공.")
    else:
        st.write("매칭 결과가 없습니다.")
    st.write("### 폐기 기록")
    if discarded:
        st.dataframe(pd.DataFrame(discarded))
    else:
        st.write("(해당 없음)")
    output = io.BytesIO()
    with pd.ExcelWriter(output, engine='openpyxl') as writer:
        if not df_out.empty:
            df_out.to_excel(writer, sheet_name='최종_커플_매칭')
    st.download_button("결과 엑셀 다운로드", data=output.getvalue(), file_name="썸매칭_결과.xlsx")

st.set_page_config(page_title="미팅파티 지망표 자동 매칭", layout="wide")
st.title("💞 미팅파티 지망표 자동 매칭 시스템")

uploaded = st.file_uploader("엑셀 파일을 업로드하세요 (.xlsx)", type=["xlsx"])

if uploaded:
    try:
        df = pd.read_excel(uploaded)
        split_idx = detect_gender_split(df)
        if split_idx is not None:
            men = df.iloc[:split_idx].copy()
            women = df.iloc[split_idx+1:].copy()
        else:
            st.error("여성 참가자 구분 행을 찾을 수 없습니다.")
            st.stop()

        men = parse_participants(men)
        women = parse_participants(women)

        candidates = build_match_candidates(men, women)
        final, discarded = resolve_duplicates(candidates)

        print_result_table(final, len(men), len(women), uploaded.name, discarded)
    except Exception as e:
        st.error(f"파일을 읽을 수 없어 분석을 진행하지 못했습니다. ({e})")
