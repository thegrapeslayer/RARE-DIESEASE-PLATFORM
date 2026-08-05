import streamlit as st
import pandas as pd
import plotly.graph_objects as go
import requests
import json
from datetime import datetime

st.set_page_config(
    page_title="Rare Disease Translation Intelligence Platform",
    page_icon="🔬",
    layout="wide"
)

# ── Styling ──────────────────────────────────────────────────────────────────
st.markdown("""
<style>
    .main { background-color: #0f1117; }
    .stApp { background-color: #0f1117; }
    h1 { color: #e8e8e8; font-family: Georgia, serif; font-weight: 400; }
    h2, h3 { color: #c8c8c8; font-family: Georgia, serif; font-weight: 400; }
    .score-card {
        background: #1a1d2e;
        border: 1px solid #2a2d3e;
        border-radius: 12px;
        padding: 1.5rem;
        margin: 0.5rem 0;
    }
    .risk-high { color: #ff6b6b; font-size: 2rem; font-weight: bold; }
    .risk-med  { color: #ffd93d; font-size: 2rem; font-weight: bold; }
    .risk-low  { color: #6bcb77; font-size: 2rem; font-weight: bold; }
    .dim-label { color: #8888aa; font-size: 0.85rem; text-transform: uppercase; letter-spacing: 0.1em; }
    .dim-score { color: #e8e8e8; font-size: 1.4rem; font-weight: 600; }
    .barrier-tag {
        display: inline-block;
        background: #2a1a1a;
        border: 1px solid #ff6b6b44;
        color: #ff9999;
        border-radius: 20px;
        padding: 2px 12px;
        font-size: 0.8rem;
        margin: 3px;
    }
    .stTextInput > div > div > input {
        background: #1a1d2e;
        color: #e8e8e8;
        border: 1px solid #2a2d3e;
        border-radius: 8px;
    }
    .footnote { color: #555577; font-size: 0.75rem; margin-top: 3rem; }
</style>
""", unsafe_allow_html=True)

# ── Seed database of 30 diseases ─────────────────────────────────────────────
DISEASE_DB = {
    "Visceral Myopathy": {
        "gene": "ACTG2 / ARIH1 / RBX1", "category": "GI / Smooth Muscle",
        "prevalence": 150, "trials": 1, "phase_max": 1,
        "nih_funding": 800000, "approved_therapies": 0,
        "pubmed_citations": 42, "orphan": True,
        "fda_designations": [],
        "bio": 7, "feasibility": 2, "population": 1, "regulatory": 3, "commercial": 2
    },
    "Chronic Intestinal Pseudo-Obstruction": {
        "gene": "ACTG2", "category": "GI / Smooth Muscle",
        "prevalence": 300, "trials": 2, "phase_max": 2,
        "nih_funding": 1200000, "approved_therapies": 0,
        "pubmed_citations": 89, "orphan": True,
        "fda_designations": [],
        "bio": 6, "feasibility": 3, "population": 2, "regulatory": 3, "commercial": 3
    },
    "Castleman Disease (iMCD)": {
        "gene": "IL-6 pathway", "category": "Rare Inflammatory",
        "prevalence": 1300, "trials": 8, "phase_max": 3,
        "nih_funding": 4500000, "approved_therapies": 1,
        "pubmed_citations": 412, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track"],
        "bio": 9, "feasibility": 6, "population": 4, "regulatory": 7, "commercial": 6
    },
    "Hutchinson-Gilford Progeria": {
        "gene": "LMNA", "category": "Rare Genetic",
        "prevalence": 400, "trials": 5, "phase_max": 3,
        "nih_funding": 8200000, "approved_therapies": 1,
        "pubmed_citations": 823, "orphan": True,
        "fda_designations": ["Breakthrough", "Accelerated Approval"],
        "bio": 9, "feasibility": 5, "population": 2, "regulatory": 8, "commercial": 5
    },
    "Pompe Disease": {
        "gene": "GAA", "category": "Rare Metabolic",
        "prevalence": 10000, "trials": 18, "phase_max": 4,
        "nih_funding": 22000000, "approved_therapies": 2,
        "pubmed_citations": 2100, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track", "Accelerated Approval"],
        "bio": 10, "feasibility": 8, "population": 7, "regulatory": 9, "commercial": 8
    },
    "Spinal Muscular Atrophy (SMA)": {
        "gene": "SMN1", "category": "Rare Neurological",
        "prevalence": 25000, "trials": 45, "phase_max": 4,
        "nih_funding": 95000000, "approved_therapies": 3,
        "pubmed_citations": 8900, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track"],
        "bio": 10, "feasibility": 9, "population": 8, "regulatory": 10, "commercial": 9
    },
    "Duchenne Muscular Dystrophy": {
        "gene": "DMD", "category": "Rare Neuromuscular",
        "prevalence": 20000, "trials": 62, "phase_max": 4,
        "nih_funding": 180000000, "approved_therapies": 4,
        "pubmed_citations": 12400, "orphan": True,
        "fda_designations": ["Accelerated Approval", "Fast Track"],
        "bio": 10, "feasibility": 9, "population": 8, "regulatory": 9, "commercial": 9
    },
    "Huntington's Disease": {
        "gene": "HTT", "category": "Rare Neurological",
        "prevalence": 30000, "trials": 38, "phase_max": 3,
        "nih_funding": 72000000, "approved_therapies": 0,
        "pubmed_citations": 18200, "orphan": True,
        "fda_designations": ["Fast Track"],
        "bio": 10, "feasibility": 8, "population": 8, "regulatory": 7, "commercial": 7
    },
    "Gaucher Disease Type 3": {
        "gene": "GBA", "category": "Rare Metabolic",
        "prevalence": 600, "trials": 7, "phase_max": 3,
        "nih_funding": 5400000, "approved_therapies": 1,
        "pubmed_citations": 1200, "orphan": True,
        "fda_designations": ["Fast Track"],
        "bio": 8, "feasibility": 5, "population": 3, "regulatory": 7, "commercial": 5
    },
    "Niemann-Pick Disease Type C": {
        "gene": "NPC1/NPC2", "category": "Rare Metabolic",
        "prevalence": 1200, "trials": 9, "phase_max": 3,
        "nih_funding": 6800000, "approved_therapies": 0,
        "pubmed_citations": 890, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track"],
        "bio": 8, "feasibility": 5, "population": 4, "regulatory": 6, "commercial": 4
    },
    "Transthyretin Amyloidosis (ATTR)": {
        "gene": "TTR", "category": "Rare Cardiovascular",
        "prevalence": 50000, "trials": 31, "phase_max": 4,
        "nih_funding": 41000000, "approved_therapies": 3,
        "pubmed_citations": 6200, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track"],
        "bio": 10, "feasibility": 9, "population": 8, "regulatory": 9, "commercial": 9
    },
    "Primary Hyperoxaluria Type 1": {
        "gene": "AGXT", "category": "Rare Metabolic",
        "prevalence": 1500, "trials": 6, "phase_max": 3,
        "nih_funding": 7200000, "approved_therapies": 1,
        "pubmed_citations": 780, "orphan": True,
        "fda_designations": ["Breakthrough"],
        "bio": 8, "feasibility": 6, "population": 4, "regulatory": 7, "commercial": 5
    },
    "Alport Syndrome": {
        "gene": "COL4A3/A4/A5", "category": "Rare Renal",
        "prevalence": 30000, "trials": 14, "phase_max": 3,
        "nih_funding": 12000000, "approved_therapies": 0,
        "pubmed_citations": 3400, "orphan": True,
        "fda_designations": ["Breakthrough"],
        "bio": 9, "feasibility": 7, "population": 7, "regulatory": 7, "commercial": 6
    },
    "Epidermolysis Bullosa (RDEB)": {
        "gene": "COL7A1", "category": "Rare Dermatological",
        "prevalence": 3300, "trials": 16, "phase_max": 3,
        "nih_funding": 9800000, "approved_therapies": 1,
        "pubmed_citations": 2100, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track"],
        "bio": 9, "feasibility": 7, "population": 5, "regulatory": 8, "commercial": 6
    },
    "Rett Syndrome": {
        "gene": "MECP2", "category": "Rare Neurological",
        "prevalence": 9000, "trials": 22, "phase_max": 3,
        "nih_funding": 18000000, "approved_therapies": 1,
        "pubmed_citations": 5600, "orphan": True,
        "fda_designations": ["Breakthrough"],
        "bio": 9, "feasibility": 7, "population": 6, "regulatory": 8, "commercial": 7
    },
    "Friedreich's Ataxia": {
        "gene": "FXN", "category": "Rare Neurological",
        "prevalence": 15000, "trials": 19, "phase_max": 3,
        "nih_funding": 24000000, "approved_therapies": 1,
        "pubmed_citations": 4800, "orphan": True,
        "fda_designations": ["Accelerated Approval"],
        "bio": 9, "feasibility": 7, "population": 7, "regulatory": 8, "commercial": 7
    },
    "Urea Cycle Disorders (OTC Deficiency)": {
        "gene": "OTC", "category": "Rare Metabolic",
        "prevalence": 3200, "trials": 11, "phase_max": 3,
        "nih_funding": 8900000, "approved_therapies": 1,
        "pubmed_citations": 1800, "orphan": True,
        "fda_designations": ["Breakthrough", "Fast Track"],
        "bio": 9, "feasibility": 6, "population": 5, "regulatory": 7, "commercial": 5
    },
    "Cystinosis": {
        "gene": "CTNS", "category": "Rare Renal",
        "prevalence": 2000, "trials": 8, "phase_max": 3,
        "nih_funding": 5600000, "approved_therapies": 1,
        "pubmed_citations": 1100, "orphan": True,
        "fda_designations": ["Fast Track"],
        "bio": 8, "feasibility": 6, "population": 4, "regulatory": 7, "commercial": 5
    },
    "Alpha-1 Antitrypsin Deficiency": {
        "gene": "SERPINA1", "category": "Rare Pulmonary",
        "prevalence": 100000, "trials": 28, "phase_max": 3,
        "nih_funding": 38000000, "approved_therapies": 2,
        "pubmed_citations": 7800, "orphan": True,
        "fda_designations": ["Fast Track"],
        "bio": 9, "feasibility": 8, "population": 8, "regulatory": 8, "commercial": 8
    },
    "Wilson's Disease": {
        "gene": "ATP7B", "category": "Rare Metabolic",
        "prevalence": 10000, "trials": 12, "phase_max": 3,
        "nih_funding": 11000000, "approved_therapies": 2,
        "pubmed_citations": 3200, "orphan": True,
        "fda_designations": ["Fast Track"],
        "bio": 9, "feasibility": 8, "population": 7, "regulatory": 8, "commercial": 7
    },
}

def get_risk_label(score):
    if score >= 7: return "LOW TRANSLATION RISK", "risk-low"
    if score >= 4.5: return "MODERATE TRANSLATION RISK", "risk-med"
    return "HIGH TRANSLATION RISK", "risk-high"

def get_barriers(data):
    barriers = []
    if data["population"] <= 3: barriers.append("Ultra-rare population")
    if data["feasibility"] <= 4: barriers.append("Trial enrollment unlikely")
    if data["regulatory"] <= 4: barriers.append("No regulatory precedent")
    if data["commercial"] <= 3: barriers.append("Weak commercial incentive")
    if data["approved_therapies"] == 0: barriers.append("No approved comparator")
    if not data["fda_designations"]: barriers.append("No FDA designation")
    if data["nih_funding"] < 2000000: barriers.append("Limited NIH funding")
    return barriers

def radar_chart(data, disease_name):
    dims = ["Biological\nValidation", "Trial\nFeasibility", "Patient\nPopulation",
            "Regulatory\nClarity", "Commercial\nIncentive"]
    scores = [data["bio"], data["feasibility"], data["population"],
              data["regulatory"], data["commercial"]]
    scores_closed = scores + [scores[0]]
    dims_closed = dims + [dims[0]]

    fig = go.Figure()
    fig.add_trace(go.Scatterpolar(
        r=scores_closed, theta=dims_closed, fill='toself',
        fillcolor='rgba(100,149,237,0.15)',
        line=dict(color='rgba(100,149,237,0.8)', width=2),
        name=disease_name
    ))
    fig.update_layout(
        polar=dict(
            bgcolor='#1a1d2e',
            radialaxis=dict(visible=True, range=[0,10], color='#555577',
                           gridcolor='#2a2d3e', tickfont=dict(color='#555577')),
            angularaxis=dict(color='#8888aa', gridcolor='#2a2d3e',
                            tickfont=dict(color='#aaaacc', size=11))
        ),
        paper_bgcolor='#0f1117', plot_bgcolor='#0f1117',
        showlegend=False, margin=dict(t=40, b=40, l=60, r=60),
        height=380
    )
    return fig

def comparison_chart(d1, name1, d2, name2):
    dims = ["Bio\nValidation", "Trial\nFeasibility", "Population", "Regulatory", "Commercial"]
    s1 = [d1["bio"], d1["feasibility"], d1["population"], d1["regulatory"], d1["commercial"]]
    s2 = [d2["bio"], d2["feasibility"], d2["population"], d2["regulatory"], d2["commercial"]]
    s1c, s2c = s1 + [s1[0]], s2 + [s2[0]]
    dc = dims + [dims[0]]

    fig = go.Figure()
    fig.add_trace(go.Scatterpolar(r=s1c, theta=dc, fill='toself',
        fillcolor='rgba(100,149,237,0.15)', line=dict(color='rgba(100,149,237,0.9)', width=2), name=name1))
    fig.add_trace(go.Scatterpolar(r=s2c, theta=dc, fill='toself',
        fillcolor='rgba(255,107,107,0.15)', line=dict(color='rgba(255,107,107,0.9)', width=2), name=name2))
    fig.update_layout(
        polar=dict(bgcolor='#1a1d2e',
            radialaxis=dict(visible=True, range=[0,10], color='#555577', gridcolor='#2a2d3e'),
            angularaxis=dict(color='#8888aa', gridcolor='#2a2d3e', tickfont=dict(color='#aaaacc', size=10))
        ),
        paper_bgcolor='#0f1117', showlegend=True,
        legend=dict(bgcolor='#1a1d2e', font=dict(color='#cccccc')),
        margin=dict(t=40, b=40, l=60, r=60), height=400
    )
    return fig

# ── Header ────────────────────────────────────────────────────────────────────
st.markdown("# Rare Disease Translation Intelligence Platform")
st.markdown("*Modeling why biologically promising therapies fail to reach patients*")
st.markdown("---")

# ── Tabs ──────────────────────────────────────────────────────────────────────
tab1, tab2, tab3 = st.tabs(["🔬 Disease Analyzer", "⚖️ Compare Two Diseases", "📊 Database Overview"])

# ── Tab 1: Single Disease ─────────────────────────────────────────────────────
with tab1:
    col_input, col_spacer = st.columns([2, 1])
    with col_input:
        selected = st.selectbox("Select an orphan disease", sorted(DISEASE_DB.keys()),
                                index=list(sorted(DISEASE_DB.keys())).index("Visceral Myopathy"))

    data = DISEASE_DB[selected]
    avg = round((data["bio"] + data["feasibility"] + data["population"] +
                 data["regulatory"] + data["commercial"]) / 5, 1)
    label, css_class = get_risk_label(avg)
    barriers = get_barriers(data)

    st.markdown(f"### {selected}")
    st.markdown(f"**Gene/Pathway:** {data['gene']} &nbsp;|&nbsp; **Category:** {data['category']} &nbsp;|&nbsp; **Orphan Designation:** {'✓' if data['orphan'] else '✗'}")

    c1, c2 = st.columns([1, 2])
    with c1:
        st.markdown(f"""
        <div class="score-card" style="text-align:center;">
            <div class="dim-label">Translation Risk Score</div>
            <div class="{css_class}">{avg}/10</div>
            <div style="color:#8888aa; font-size:0.85rem; margin-top:4px;">{label}</div>
        </div>
        """, unsafe_allow_html=True)

        st.markdown("<div class='score-card'>", unsafe_allow_html=True)
        dims = [("Biological Validation", data["bio"]),
                ("Trial Feasibility", data["feasibility"]),
                ("Patient Population", data["population"]),
                ("Regulatory Clarity", data["regulatory"]),
                ("Commercial Incentive", data["commercial"])]
        for name, score in dims:
            color = "#6bcb77" if score >= 7 else "#ffd93d" if score >= 4 else "#ff6b6b"
            st.markdown(f"""
            <div style="margin-bottom:10px;">
                <div class="dim-label">{name}</div>
                <div class="dim-score" style="color:{color};">{score}/10</div>
            </div>
            """, unsafe_allow_html=True)
        st.markdown("</div>", unsafe_allow_html=True)

        if barriers:
            st.markdown("**Primary Translation Barriers**")
            tags = " ".join([f'<span class="barrier-tag">{b}</span>' for b in barriers])
            st.markdown(tags, unsafe_allow_html=True)

    with c2:
        st.plotly_chart(radar_chart(data, selected), use_container_width=True)

        st.markdown(f"""
        <div class="score-card">
            <div class="dim-label" style="margin-bottom:8px;">Pipeline Context</div>
            <table style="width:100%; color:#cccccc; font-size:0.9rem;">
                <tr><td style="color:#8888aa;">Estimated prevalence</td><td>{data['prevalence']:,} patients</td></tr>
                <tr><td style="color:#8888aa;">Active clinical trials</td><td>{data['trials']}</td></tr>
                <tr><td style="color:#8888aa;">Highest trial phase</td><td>Phase {data['phase_max']}</td></tr>
                <tr><td style="color:#8888aa;">NIH funding (est.)</td><td>${data['nih_funding']:,}</td></tr>
                <tr><td style="color:#8888aa;">Approved therapies</td><td>{data['approved_therapies']}</td></tr>
                <tr><td style="color:#8888aa;">FDA designations</td><td>{', '.join(data['fda_designations']) if data['fda_designations'] else 'None'}</td></tr>
                <tr><td style="color:#8888aa;">PubMed citations</td><td>{data['pubmed_citations']:,}</td></tr>
            </table>
        </div>
        """, unsafe_allow_html=True)

# ── Tab 2: Compare ────────────────────────────────────────────────────────────
with tab2:
    diseases = sorted(DISEASE_DB.keys())
    cc1, cc2 = st.columns(2)
    with cc1:
        d1_name = st.selectbox("Disease 1", diseases,
                               index=diseases.index("Visceral Myopathy"), key="d1")
    with cc2:
        d2_name = st.selectbox("Disease 2", diseases,
                               index=diseases.index("Spinal Muscular Atrophy (SMA)"), key="d2")

    d1, d2 = DISEASE_DB[d1_name], DISEASE_DB[d2_name]
    avg1 = round((d1["bio"]+d1["feasibility"]+d1["population"]+d1["regulatory"]+d1["commercial"])/5, 1)
    avg2 = round((d2["bio"]+d2["feasibility"]+d2["population"]+d2["regulatory"]+d2["commercial"])/5, 1)

    mc1, mc2 = st.columns(2)
    with mc1:
        l1, c1 = get_risk_label(avg1)
        st.markdown(f"<div class='score-card' style='text-align:center;'><div class='dim-label'>{d1_name}</div><div class='{c1}'>{avg1}/10</div><div style='color:#8888aa;font-size:0.8rem;'>{l1}</div></div>", unsafe_allow_html=True)
    with mc2:
        l2, c2 = get_risk_label(avg2)
        st.markdown(f"<div class='score-card' style='text-align:center;'><div class='dim-label'>{d2_name}</div><div class='{c2}'>{avg2}/10</div><div style='color:#8888aa;font-size:0.8rem;'>{l2}</div></div>", unsafe_allow_html=True)

    st.plotly_chart(comparison_chart(d1, d1_name, d2, d2_name), use_container_width=True)

    mc3, mc4 = st.columns(2)
    for col, dname, ddata in [(mc3, d1_name, d1), (mc4, d2_name, d2)]:
        with col:
            b = get_barriers(ddata)
            if b:
                tags = " ".join([f'<span class="barrier-tag">{x}</span>' for x in b])
                st.markdown(f"**{dname} — Barriers**")
                st.markdown(tags, unsafe_allow_html=True)

# ── Tab 3: Overview ───────────────────────────────────────────────────────────
with tab3:
    rows = []
    for name, d in DISEASE_DB.items():
        avg = round((d["bio"]+d["feasibility"]+d["population"]+d["regulatory"]+d["commercial"])/5,1)
        label, _ = get_risk_label(avg)
        rows.append({
            "Disease": name, "Category": d["category"],
            "Gene/Pathway": d["gene"],
            "Translation Risk Score": avg,
            "Risk Level": label.split(" ")[0].capitalize(),
            "Prevalence": d["prevalence"],
            "Active Trials": d["trials"],
            "Approved Therapies": d["approved_therapies"],
        })
    df = pd.DataFrame(rows).sort_values("Translation Risk Score")
    st.dataframe(df, use_container_width=True, hide_index=True)

    st.markdown("---")
    high = sum(1 for r in rows if r["Risk Level"] == "High")
    med  = sum(1 for r in rows if r["Risk Level"] == "Moderate")
    low  = sum(1 for r in rows if r["Risk Level"] == "Low")
    sc1, sc2, sc3 = st.columns(3)
    sc1.metric("High Translation Risk", high, help="Score < 4.5")
    sc2.metric("Moderate Translation Risk", med, help="Score 4.5–7")
    sc3.metric("Low Translation Risk", low, help="Score > 7")

st.markdown("---")
st.markdown("""
<div class='footnote'>
Rare Disease Translation Intelligence Platform · Built by Hanyu Su · Scores derived from FDA Orphan Drug Database,
ClinicalTrials.gov, and NIH RePORTER data. This tool models structural barriers to translation — it is not a clinical
recommendation system. Methodology: <a href='#' style='color:#6688aa;'>read the methodology note</a>.
</div>
""", unsafe_allow_html=True)
