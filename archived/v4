from __future__ import annotations

import math
from datetime import datetime, timezone
from typing import Any
from urllib.parse import quote_plus

import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import requests
import streamlit as st
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

st.set_page_config(
    page_title="RDTI | Rare Disease Translation Intelligence",
    page_icon="🧬",
    layout="wide",
    initial_sidebar_state="expanded",
)

# -----------------------------------------------------------------------------
# Theme
# -----------------------------------------------------------------------------
st.markdown(
    """
<style>
:root {
    --ink: #24343b;
    --muted: #667980;
    --paper: #f7faf8;
    --card: #ffffff;
    --line: #dfe9e4;
    --sage: #6f9382;
    --sage-dark: #486d5d;
    --blue: #6f91a5;
    --rose: #b77b78;
    --amber: #b28a51;
}
.stApp { background: var(--paper); color: var(--ink); }
.block-container { padding-top: 2rem; padding-bottom: 3rem; max-width: 1480px; }
h1, h2, h3 { color: var(--ink); font-family: Inter, ui-sans-serif, system-ui, sans-serif; letter-spacing: -0.025em; }
h1 { font-weight: 650; }
p, label, .stMarkdown { color: var(--ink); }
[data-testid="stSidebar"] { background: #eef5f1; border-right: 1px solid var(--line); }
[data-testid="stMetric"] {
    background: var(--card); border: 1px solid var(--line); border-radius: 16px;
    padding: 16px 18px; box-shadow: 0 3px 16px rgba(54, 77, 67, 0.04);
}
[data-testid="stMetricLabel"] { color: var(--muted); }
[data-testid="stMetricValue"] { color: var(--ink); }
.soft-card {
    background: var(--card); border: 1px solid var(--line); border-radius: 18px;
    padding: 1.25rem 1.35rem; box-shadow: 0 5px 22px rgba(54, 77, 67, 0.045);
    margin-bottom: 0.8rem;
}
.eyebrow { color: var(--sage-dark); text-transform: uppercase; font-size: .73rem; letter-spacing: .11em; font-weight: 700; }
.hero-copy { color: var(--muted); font-size: 1.04rem; max-width: 850px; margin-top: -.35rem; }
.score-number { font-size: 2.45rem; line-height: 1; font-weight: 720; color: var(--ink); }
.score-label { color: var(--muted); font-size: .85rem; margin-top: .35rem; }
.pill {
    display: inline-block; border-radius: 999px; padding: 5px 10px; margin: 3px 4px 3px 0;
    font-size: .78rem; background: #edf3f0; color: #456558; border: 1px solid #d8e5df;
}
.pill-risk { background: #f8eeee; color: #8a5351; border-color: #ecd6d5; }
.source-badge { display:inline-block; padding: 4px 8px; border-radius: 8px; font-size: .72rem; margin-right: 5px; }
.source-live { background:#e7f3ed; color:#3d6d58; }
.source-seed { background:#f2f0e8; color:#786b44; }
.small-note { color: var(--muted); font-size: .78rem; line-height: 1.45; }
hr { border-color: var(--line); }
.stButton > button, .stDownloadButton > button {
    border-radius: 11px; border: 1px solid #bfcfc7; background: #ffffff; color: var(--sage-dark);
}
.stButton > button:hover, .stDownloadButton > button:hover { border-color: var(--sage); color: #355848; }
div[data-baseweb="select"] > div { background: white; border-color: var(--line); }
.stTabs [data-baseweb="tab-list"] { gap: .35rem; }
.stTabs [data-baseweb="tab"] { background:#edf3f0; border-radius:10px 10px 0 0; padding:.55rem 1rem; }
.stTabs [aria-selected="true"] { background:#ffffff; color:var(--sage-dark); }
</style>
""",
    unsafe_allow_html=True,
)

# -----------------------------------------------------------------------------
# Disease catalog: 100 orphan and rare diseases.
# Seed values are a transparent modeling baseline. Live sources replace selected
# evidence fields on demand.
# -----------------------------------------------------------------------------
CATALOG = [
    ("Visceral Myopathy", "ACTG2 / MYH11 / LMOD1", "Gastrointestinal", 150),
    ("Chronic Intestinal Pseudo-Obstruction", "ACTG2 / FLNA", "Gastrointestinal", 300),
    ("Idiopathic Multicentric Castleman Disease", "IL-6 pathway", "Immune / Inflammatory", 1300),
    ("Hutchinson-Gilford Progeria Syndrome", "LMNA", "Genetic / Premature aging", 400),
    ("Pompe Disease", "GAA", "Metabolic", 10000),
    ("Spinal Muscular Atrophy", "SMN1", "Neuromuscular", 25000),
    ("Duchenne Muscular Dystrophy", "DMD", "Neuromuscular", 20000),
    ("Huntington Disease", "HTT", "Neurological", 30000),
    ("Gaucher Disease", "GBA1", "Lysosomal", 10000),
    ("Niemann-Pick Disease Type C", "NPC1 / NPC2", "Lysosomal", 1200),
    ("Hereditary Transthyretin Amyloidosis", "TTR", "Cardiovascular / Neurological", 50000),
    ("Primary Hyperoxaluria Type 1", "AGXT", "Renal / Metabolic", 1500),
    ("Alport Syndrome", "COL4A3 / COL4A4 / COL4A5", "Renal", 30000),
    ("Recessive Dystrophic Epidermolysis Bullosa", "COL7A1", "Dermatological", 3300),
    ("Rett Syndrome", "MECP2", "Neurological", 9000),
    ("Friedreich Ataxia", "FXN", "Neurological", 15000),
    ("Ornithine Transcarbamylase Deficiency", "OTC", "Metabolic", 3200),
    ("Cystinosis", "CTNS", "Renal / Lysosomal", 2000),
    ("Alpha-1 Antitrypsin Deficiency", "SERPINA1", "Pulmonary / Hepatic", 100000),
    ("Wilson Disease", "ATP7B", "Metabolic / Hepatic", 10000),
    ("Fabry Disease", "GLA", "Lysosomal", 40000),
    ("Mucopolysaccharidosis Type I", "IDUA", "Lysosomal", 5000),
    ("Mucopolysaccharidosis Type II", "IDS", "Lysosomal", 2000),
    ("Mucopolysaccharidosis Type III", "SGSH / NAGLU / HGSNAT / GNS", "Lysosomal", 7000),
    ("Mucopolysaccharidosis Type IV A", "GALNS", "Lysosomal / Skeletal", 3000),
    ("Mucopolysaccharidosis Type VI", "ARSB", "Lysosomal", 1100),
    ("Krabbe Disease", "GALC", "Leukodystrophy", 3000),
    ("Metachromatic Leukodystrophy", "ARSA", "Leukodystrophy", 4000),
    ("X-linked Adrenoleukodystrophy", "ABCD1", "Leukodystrophy", 18000),
    ("Canavan Disease", "ASPA", "Leukodystrophy", 1000),
    ("Alexander Disease", "GFAP", "Leukodystrophy", 500),
    ("Pelizaeus-Merzbacher Disease", "PLP1", "Leukodystrophy", 2500),
    ("Batten Disease CLN2", "TPP1", "Lysosomal / Neurological", 1200),
    ("Batten Disease CLN3", "CLN3", "Lysosomal / Neurological", 3000),
    ("Tay-Sachs Disease", "HEXA", "Lysosomal", 1000),
    ("Sandhoff Disease", "HEXB", "Lysosomal", 500),
    ("GM1 Gangliosidosis", "GLB1", "Lysosomal", 1200),
    ("MPS VII (Sly Syndrome)", "GUSB", "Lysosomal", 200),
    ("Acid Sphingomyelinase Deficiency", "SMPD1", "Lysosomal", 1500),
    ("Wolman Disease / LAL Deficiency", "LIPA", "Lysosomal / Metabolic", 1200),
    ("Phenylketonuria", "PAH", "Metabolic", 16000),
    ("Maple Syrup Urine Disease", "BCKDHA / BCKDHB / DBT", "Metabolic", 2500),
    ("Methylmalonic Acidemia", "MUT / MMAA / MMAB", "Metabolic", 3000),
    ("Propionic Acidemia", "PCCA / PCCB", "Metabolic", 2500),
    ("Homocystinuria", "CBS", "Metabolic", 4000),
    ("Tyrosinemia Type I", "FAH", "Metabolic / Hepatic", 1200),
    ("Hereditary Fructose Intolerance", "ALDOB", "Metabolic", 6000),
    ("Glutaric Acidemia Type I", "GCDH", "Metabolic", 1000),
    ("Medium-chain Acyl-CoA Dehydrogenase Deficiency", "ACADM", "Metabolic", 15000),
    ("Very Long-chain Acyl-CoA Dehydrogenase Deficiency", "ACADVL", "Metabolic", 5000),
    ("Long-chain 3-Hydroxyacyl-CoA Dehydrogenase Deficiency", "HADHA / HADHB", "Metabolic", 2000),
    ("Glycogen Storage Disease Type I", "G6PC / SLC37A4", "Metabolic", 6000),
    ("Glycogen Storage Disease Type III", "AGL", "Metabolic", 5000),
    ("Glycogen Storage Disease Type V", "PYGM", "Metabolic / Neuromuscular", 8000),
    ("Mitochondrial Encephalomyopathy, Lactic Acidosis, and Stroke-like Episodes", "MT-TL1", "Mitochondrial", 10000),
    ("Leber Hereditary Optic Neuropathy", "MT-ND1 / MT-ND4 / MT-ND6", "Mitochondrial / Ophthalmic", 12000),
    ("Thymidine Kinase 2 Deficiency", "TK2", "Mitochondrial", 300),
    ("POLG-related Disorders", "POLG", "Mitochondrial", 7000),
    ("Barth Syndrome", "TAZ", "Mitochondrial / Cardiovascular", 250),
    ("Dravet Syndrome", "SCN1A", "Neurological", 35000),
    ("Lennox-Gastaut Syndrome", "Multiple", "Neurological", 50000),
    ("CDKL5 Deficiency Disorder", "CDKL5", "Neurological", 15000),
    ("Angelman Syndrome", "UBE3A", "Neurological", 25000),
    ("Prader-Willi Syndrome", "15q11-q13", "Endocrine / Neurological", 20000),
    ("Fragile X Syndrome", "FMR1", "Neurological", 100000),
    ("Tuberous Sclerosis Complex", "TSC1 / TSC2", "Neurological / Multisystem", 50000),
    ("Neurofibromatosis Type 1", "NF1", "Neurological / Oncology", 100000),
    ("Neurofibromatosis Type 2", "NF2", "Neurological / Oncology", 12000),
    ("Amyotrophic Lateral Sclerosis", "C9orf72 / SOD1 / FUS / TARDBP", "Neurological", 30000),
    ("Multiple System Atrophy", "Multiple", "Neurological", 15000),
    ("Progressive Supranuclear Palsy", "MAPT-associated", "Neurological", 20000),
    ("Hereditary Spastic Paraplegia", "Multiple", "Neurological", 20000),
    ("Charcot-Marie-Tooth Disease Type 1A", "PMP22", "Neuromuscular", 100000),
    ("Myotonic Dystrophy Type 1", "DMPK", "Neuromuscular", 40000),
    ("Facioscapulohumeral Muscular Dystrophy", "DUX4", "Neuromuscular", 30000),
    ("Limb-Girdle Muscular Dystrophy R2", "DYSF", "Neuromuscular", 8000),
    ("GNE Myopathy", "GNE", "Neuromuscular", 3000),
    ("Congenital Myasthenic Syndrome", "Multiple", "Neuromuscular", 5000),
    ("Generalized Myasthenia Gravis", "AChR / MuSK / LRP4", "Autoimmune / Neuromuscular", 70000),
    ("Paroxysmal Nocturnal Hemoglobinuria", "PIGA", "Hematological", 6000),
    ("Atypical Hemolytic Uremic Syndrome", "Complement pathway", "Hematological / Renal", 5000),
    ("Immune Thrombocytopenia", "Autoimmune", "Hematological", 100000),
    ("Sickle Cell Disease", "HBB", "Hematological", 100000),
    ("Beta Thalassemia", "HBB", "Hematological", 60000),
    ("Hemophilia A", "F8", "Hematological", 25000),
    ("Hemophilia B", "F9", "Hematological", 7000),
    ("Congenital Thrombotic Thrombocytopenic Purpura", "ADAMTS13", "Hematological", 1000),
    ("Hereditary Angioedema", "SERPING1 / F12", "Immune / Vascular", 10000),
    ("Cryopyrin-Associated Periodic Syndromes", "NLRP3", "Autoinflammatory", 1000),
    ("Familial Mediterranean Fever", "MEFV", "Autoinflammatory", 100000),
    ("Deficiency of Adenosine Deaminase 2", "ADA2", "Autoinflammatory / Vascular", 600),
    ("WHIM Syndrome", "CXCR4", "Immunodeficiency", 1000),
    ("Wiskott-Aldrich Syndrome", "WAS", "Immunodeficiency", 5000),
    ("Severe Combined Immunodeficiency", "Multiple", "Immunodeficiency", 2000),
    ("Chronic Granulomatous Disease", "CYBB / NCF1 / NCF2", "Immunodeficiency", 2500),
    ("Pulmonary Arterial Hypertension", "BMPR2 / pathway", "Cardiopulmonary", 50000),
    ("Idiopathic Pulmonary Fibrosis", "Multiple", "Pulmonary", 100000),
    ("Lymphangioleiomyomatosis", "TSC1 / TSC2", "Pulmonary", 3500),
    ("Cystic Fibrosis", "CFTR", "Pulmonary", 40000),
    ("Primary Ciliary Dyskinesia", "Multiple", "Pulmonary", 25000),
]

assert len(CATALOG) == 100, f"Catalog must contain 100 diseases, found {len(CATALOG)}"

ADVANCED = {
    "Pompe Disease", "Spinal Muscular Atrophy", "Duchenne Muscular Dystrophy",
    "Hereditary Transthyretin Amyloidosis", "Cystic Fibrosis", "Sickle Cell Disease",
    "Hemophilia A", "Hemophilia B", "Gaucher Disease", "Fabry Disease",
    "Phenylketonuria", "Paroxysmal Nocturnal Hemoglobinuria", "Hereditary Angioedema",
}
EMERGING = {
    "Rett Syndrome", "Friedreich Ataxia", "Recessive Dystrophic Epidermolysis Bullosa",
    "Niemann-Pick Disease Type C", "Primary Hyperoxaluria Type 1", "Dravet Syndrome",
    "CDKL5 Deficiency Disorder", "Angelman Syndrome", "Metachromatic Leukodystrophy",
    "Batten Disease CLN2", "Beta Thalassemia", "Atypical Hemolytic Uremic Syndrome",
    "Pulmonary Arterial Hypertension", "Generalized Myasthenia Gravis",
}


def clamp(value: float, low: float = 1.0, high: float = 10.0) -> float:
    return round(max(low, min(high, value)), 1)


def build_seed_database() -> dict[str, dict[str, Any]]:
    db: dict[str, dict[str, Any]] = {}
    for i, (name, gene, category, prevalence) in enumerate(CATALOG):
        pop_score = clamp(1.4 + 1.42 * math.log10(max(prevalence, 100) / 100))
        maturity = 8.5 if name in ADVANCED else 6.7 if name in EMERGING else 4.6 + ((i * 7) % 13) / 10
        bio = clamp(maturity + (0.5 if gene != "Multiple" else -0.3))
        feasibility = clamp((maturity + pop_score) / 2 - (0.7 if prevalence < 1000 else 0))
        regulatory = clamp(maturity - 0.2)
        commercial = clamp((pop_score * 0.55) + (maturity * 0.45) - 0.3)
        trials = max(0, int((maturity - 3.5) * 4 + math.log10(max(prevalence, 100)) * 2 - 5))
        approved = 2 if name in ADVANCED else 1 if name in EMERGING else 0
        phase_max = 4 if name in ADVANCED else 3 if name in EMERGING else (2 if maturity >= 5.3 else 1)
        citations = int(25 * (maturity ** 2) * max(1, math.log10(max(prevalence, 100))))
        funding = int((maturity ** 2) * max(150_000, math.log10(max(prevalence, 100)) * 210_000))
        db[name] = {
            "gene": gene,
            "category": category,
            "prevalence": prevalence,
            "bio": bio,
            "feasibility": feasibility,
            "population": pop_score,
            "regulatory": regulatory,
            "commercial": commercial,
            "trials": trials,
            "phase_max": phase_max,
            "nih_funding": funding,
            "approved_therapies": approved,
            "pubmed_citations": citations,
            "orphan": True,
            "data_source": "Modeled baseline",
            "last_refreshed": None,
        }

    # Project-specific overrides and better-known benchmarks.
    db["Visceral Myopathy"].update(
        bio=7.0, feasibility=2.0, population=1.0, regulatory=3.0, commercial=2.0,
        trials=1, phase_max=1, nih_funding=800_000, approved_therapies=0,
        pubmed_citations=42, gene="ACTG2 / MYH11 / LMOD1 / ARIH1 / RBX1"
    )
    db["Chronic Intestinal Pseudo-Obstruction"].update(
        bio=6.0, feasibility=3.0, population=2.0, regulatory=3.0, commercial=3.0,
        trials=2, phase_max=2, nih_funding=1_200_000, approved_therapies=0,
        pubmed_citations=89,
    )
    return db


DISEASE_DB = build_seed_database()

# -----------------------------------------------------------------------------
# APIs
# -----------------------------------------------------------------------------

def make_session() -> requests.Session:
    retry = Retry(total=2, backoff_factor=0.35, status_forcelist=(429, 500, 502, 503, 504), allowed_methods=("GET", "POST"))
    session = requests.Session()
    session.mount("https://", HTTPAdapter(max_retries=retry))
    session.headers.update({"User-Agent": "RDTI-Research-App/1.0 (educational rare-disease analytics)"})
    return session


HTTP = make_session()


@st.cache_data(ttl=60 * 60 * 12, show_spinner=False)
def fetch_clinical_trials(disease: str) -> dict[str, Any]:
    params = {
        "query.cond": disease,
        "pageSize": 100,
        "format": "json",
        "countTotal": "true",
        "fields": "NCTId,BriefTitle,OverallStatus,Phase,StartDate,CompletionDate,LeadSponsorName",
    }
    response = HTTP.get("https://clinicaltrials.gov/api/v2/studies", params=params, timeout=18)
    response.raise_for_status()
    payload = response.json()
    studies = payload.get("studies", [])
    active_statuses = {"RECRUITING", "NOT_YET_RECRUITING", "ACTIVE_NOT_RECRUITING", "ENROLLING_BY_INVITATION"}
    active = 0
    max_phase = 0
    recent = []
    phase_map = {"EARLY_PHASE1": 1, "PHASE1": 1, "PHASE2": 2, "PHASE3": 3, "PHASE4": 4}
    for study in studies:
        protocol = study.get("protocolSection", {})
        ident = protocol.get("identificationModule", {})
        status = protocol.get("statusModule", {})
        design = protocol.get("designModule", {})
        sponsor = protocol.get("sponsorCollaboratorsModule", {}).get("leadSponsor", {})
        overall = status.get("overallStatus", "UNKNOWN")
        if overall in active_statuses:
            active += 1
        phases = design.get("phases", []) or []
        phase_values = [phase_map.get(p, 0) for p in phases]
        if phase_values:
            max_phase = max(max_phase, *phase_values)
        if len(recent) < 8:
            recent.append({
                "NCT ID": ident.get("nctId", ""),
                "Study": ident.get("briefTitle", "Untitled study"),
                "Status": overall.replace("_", " ").title(),
                "Phase": ", ".join(p.replace("PHASE", "Phase ").replace("EARLY_", "Early ") for p in phases) or "Not applicable",
                "Sponsor": sponsor.get("name", "Not listed"),
            })
    return {"total": int(payload.get("totalCount", len(studies))), "active": active, "phase_max": max_phase, "studies": recent}


@st.cache_data(ttl=60 * 60 * 24, show_spinner=False)
def fetch_pubmed_count(disease: str) -> int:
    response = HTTP.get(
        "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi",
        params={"db": "pubmed", "term": f'"{disease}"[Title/Abstract]', "retmode": "json", "retmax": 0},
        timeout=15,
    )
    response.raise_for_status()
    return int(response.json()["esearchresult"]["count"])


@st.cache_data(ttl=60 * 60 * 24, show_spinner=False)
def fetch_nih_funding(disease: str) -> dict[str, int]:
    current_year = datetime.now(timezone.utc).year
    payload = {
        "criteria": {
            "advanced_text_search": {"operator": "and", "search_field": "all", "search_text": disease},
            "fiscal_years": list(range(current_year - 4, current_year + 1)),
        },
        "include_fields": ["ProjectNum", "AwardAmount", "FiscalYear"],
        "offset": 0,
        "limit": 500,
    }
    response = HTTP.post("https://api.reporter.nih.gov/v2/projects/search", json=payload, timeout=25)
    response.raise_for_status()
    results = response.json().get("results", [])
    total = sum(int(row.get("award_amount") or 0) for row in results)
    return {"funding": total, "projects": len(results)}


def _friendly_error(exc: Exception) -> str:
    """Turn a raw exception into a short, human-readable reason."""
    if isinstance(exc, requests.exceptions.Timeout):
        return "timed out"
    if isinstance(exc, requests.exceptions.ConnectionError):
        return "no network access"
    if isinstance(exc, requests.exceptions.HTTPError):
        return f"server returned an error ({getattr(exc.response, 'status_code', '?')})"
    if isinstance(exc, (KeyError, ValueError, TypeError)):
        return "unexpected response format"
    return type(exc).__name__


def refresh_live_data(disease: str, seed: dict[str, Any]) -> tuple[dict[str, Any], list[str]]:
    """Try each public source independently. Any source that fails simply
    keeps its modeled baseline value instead of breaking the page, so the
    app always has something sensible to show even with no internet access."""
    data = dict(seed)
    errors: list[str] = []
    live_fields = []
    try:
        trials = fetch_clinical_trials(disease)
        data["trials"] = trials["active"]
        data["registered_trials"] = trials["total"]
        data["phase_max"] = trials["phase_max"] or data["phase_max"]
        data["trial_records"] = trials["studies"]
        live_fields.append("ClinicalTrials.gov")
    except Exception as exc:
        errors.append(f"ClinicalTrials.gov ({_friendly_error(exc)})")
    try:
        data["pubmed_citations"] = fetch_pubmed_count(disease)
        live_fields.append("PubMed")
    except Exception as exc:
        errors.append(f"PubMed ({_friendly_error(exc)})")
    try:
        nih = fetch_nih_funding(disease)
        if nih["funding"] > 0:
            data["nih_funding"] = nih["funding"]
        data["nih_projects"] = nih["projects"]
        live_fields.append("NIH RePORTER")
    except Exception as exc:
        errors.append(f"NIH RePORTER ({_friendly_error(exc)})")
    data["data_source"] = " + ".join(live_fields) if live_fields else "Modeled baseline"
    data["last_refreshed"] = datetime.now().astimezone().strftime("%b %d, %Y at %I:%M %p")
    return data, errors

# -----------------------------------------------------------------------------
# Scoring and charts
# -----------------------------------------------------------------------------
WEIGHTS = {"bio": 0.25, "feasibility": 0.25, "population": 0.15, "regulatory": 0.20, "commercial": 0.15}


def opportunity_score(data: dict[str, Any]) -> float:
    return round(sum(data[key] * weight for key, weight in WEIGHTS.items()), 1)


def risk_label(score: float) -> tuple[str, str]:
    if score >= 7.0:
        return "Lower translation risk", "#557e69"
    if score >= 4.5:
        return "Moderate translation risk", "#aa7f43"
    return "Higher translation risk", "#a96764"


def barriers(data: dict[str, Any]) -> list[str]:
    result = []
    if data["population"] <= 3.0: result.append("Ultra-rare enrollment base")
    if data["feasibility"] <= 4.0: result.append("Difficult trial execution")
    if data["regulatory"] <= 4.0: result.append("Limited regulatory precedent")
    if data["commercial"] <= 3.5: result.append("Weak conventional market incentive")
    if data["approved_therapies"] == 0: result.append("No modeled approved comparator")
    if data["trials"] <= 2: result.append("Sparse active pipeline")
    return result


def radar_chart(data: dict[str, Any], name: str) -> go.Figure:
    labels = ["Biological validation", "Trial feasibility", "Patient population", "Regulatory clarity", "Commercial incentive"]
    values = [data["bio"], data["feasibility"], data["population"], data["regulatory"], data["commercial"]]
    fig = go.Figure(go.Scatterpolar(
        r=values + [values[0]], theta=labels + [labels[0]], fill="toself",
        fillcolor="rgba(111,147,130,0.20)", line=dict(color="#6f9382", width=2), name=name,
    ))
    fig.update_layout(
        polar=dict(
            bgcolor="#ffffff",
            radialaxis=dict(range=[0, 10], tickvals=[2, 4, 6, 8, 10], gridcolor="#dfe9e4", linecolor="#dfe9e4", tickfont=dict(color="#7d8d85")),
            angularaxis=dict(gridcolor="#dfe9e4", linecolor="#dfe9e4", tickfont=dict(color="#52665d", size=11)),
        ),
        paper_bgcolor="rgba(0,0,0,0)", margin=dict(t=35, b=35, l=65, r=65), height=405, showlegend=False,
    )
    return fig


def compare_chart(a: dict[str, Any], a_name: str, b: dict[str, Any], b_name: str) -> go.Figure:
    labels = ["Biology", "Trial feasibility", "Population", "Regulatory", "Commercial"]
    fig = go.Figure()
    for data, name, color, fill in [
        (a, a_name, "#6f9382", "rgba(111,147,130,.16)"),
        (b, b_name, "#7f95aa", "rgba(127,149,170,.14)"),
    ]:
        vals = [data["bio"], data["feasibility"], data["population"], data["regulatory"], data["commercial"]]
        fig.add_trace(go.Scatterpolar(r=vals + [vals[0]], theta=labels + [labels[0]], fill="toself", fillcolor=fill, line=dict(color=color, width=2), name=name))
    fig.update_layout(
        polar=dict(bgcolor="#fff", radialaxis=dict(range=[0, 10], gridcolor="#dfe9e4"), angularaxis=dict(gridcolor="#dfe9e4")),
        paper_bgcolor="rgba(0,0,0,0)", height=430, margin=dict(t=30, b=30, l=60, r=60),
        legend=dict(orientation="h", y=-0.08, x=0.5, xanchor="center"),
    )
    return fig


def source_badge(data: dict[str, Any]) -> str:
    live = data["data_source"] != "Modeled baseline"
    cls = "source-live" if live else "source-seed"
    text = f"Live: {data['data_source']}" if live else "Modeled baseline"
    return f'<span class="source-badge {cls}">{text}</span>'

# -----------------------------------------------------------------------------
# Sidebar
# -----------------------------------------------------------------------------
with st.sidebar:
    st.markdown("### RDTI")
    st.caption("Rare Disease Translation Initiative")
    st.markdown("---")
    categories = sorted({d[2] for d in CATALOG})
    category_filter = st.multiselect("Filter categories", categories, placeholder="All categories")
    min_score = st.slider("Minimum translation score", 1.0, 10.0, 1.0, 0.5)
    st.markdown("---")
    st.markdown("**Evidence connections**")
    st.caption("ClinicalTrials.gov API v2")
    st.caption("NCBI PubMed E-utilities")
    st.caption("NIH RePORTER API v2")
    st.caption("FDA orphan designation database, linked for verification")
    st.markdown("---")
    st.caption("Scores are research prioritization signals, not clinical recommendations or validated investment advice.")

# -----------------------------------------------------------------------------
# Header and state
# -----------------------------------------------------------------------------
st.markdown('<div class="eyebrow">Rare Disease Translation Initiative</div>', unsafe_allow_html=True)
st.title("Translation Intelligence Platform")
st.markdown(
    '<div class="hero-copy">Explore where rare-disease programs are most likely to stall, compare diseases across five translation dimensions, and refresh core evidence from public biomedical APIs.</div>',
    unsafe_allow_html=True,
)

if "live_data" not in st.session_state:
    st.session_state.live_data = {}
if "api_errors" not in st.session_state:
    st.session_state.api_errors = {}

filtered_names = []
for name, data in DISEASE_DB.items():
    if category_filter and data["category"] not in category_filter:
        continue
    if opportunity_score(data) < min_score:
        continue
    filtered_names.append(name)
filtered_names = sorted(filtered_names)
if not filtered_names:
    st.warning("No diseases match the current sidebar filters.")
    st.stop()

analyze_tab, compare_tab, portfolio_tab, methods_tab = st.tabs([
    "Disease analyzer", "Compare", "Portfolio", "Methodology"
])

with analyze_tab:
    top1, top2 = st.columns([3, 1])
    with top1:
        default_index = filtered_names.index("Visceral Myopathy") if "Visceral Myopathy" in filtered_names else 0
        selected = st.selectbox("Choose an orphan disease", filtered_names, index=default_index)
    with top2:
        st.write("")
        st.write("")
        refresh = st.button("Refresh live evidence", use_container_width=True, type="primary")

    if refresh:
        with st.spinner("Querying public biomedical sources..."):
            live, errs = refresh_live_data(selected, DISEASE_DB[selected])
            st.session_state.live_data[selected] = live
            st.session_state.api_errors[selected] = errs

    data = st.session_state.live_data.get(selected, DISEASE_DB[selected])
    score = opportunity_score(data)
    risk, risk_color = risk_label(score)

    st.markdown(f"## {selected}")
    st.markdown(
        f"**Gene or pathway:** {data['gene']} &nbsp;&nbsp; • &nbsp;&nbsp; **Category:** {data['category']} &nbsp;&nbsp; • &nbsp;&nbsp; {source_badge(data)}",
        unsafe_allow_html=True,
    )
    if data.get("last_refreshed"):
        st.caption(f"Last live refresh: {data['last_refreshed']}")
    if st.session_state.api_errors.get(selected):
        st.info("Some sources were unavailable, so the app retained modeled values for those fields: " + ", ".join(st.session_state.api_errors[selected]))

    m1, m2, m3, m4, m5 = st.columns(5)
    m1.metric("Translation score", f"{score}/10")
    m2.metric("Estimated prevalence", f"{data['prevalence']:,}")
    m3.metric("Active trials", f"{data['trials']:,}")
    m4.metric("PubMed records", f"{data['pubmed_citations']:,}")
    m5.metric("NIH funding, 5y", f"${data['nih_funding']/1_000_000:.1f}M")

    left, right = st.columns([1.05, 1.55], gap="large")
    with left:
        st.markdown(
            f"""
<div class="soft-card">
  <div class="eyebrow">Modeled translation outlook</div>
  <div class="score-number" style="color:{risk_color};">{score}</div>
  <div class="score-label">{risk}</div>
</div>
""",
            unsafe_allow_html=True,
        )
        st.markdown("#### Dimension scores")
        dimension_df = pd.DataFrame({
            "Dimension": ["Biological validation", "Trial feasibility", "Patient population", "Regulatory clarity", "Commercial incentive"],
            "Score": [data["bio"], data["feasibility"], data["population"], data["regulatory"], data["commercial"]],
        })
        st.dataframe(dimension_df, hide_index=True, use_container_width=True, height=212)
        b = barriers(data)
        if b:
            st.markdown("#### Priority barriers")
            st.markdown("".join(f'<span class="pill pill-risk">{item}</span>' for item in b), unsafe_allow_html=True)
    with right:
        st.plotly_chart(radar_chart(data, selected), use_container_width=True, config={"displayModeBar": False})

    st.markdown("#### Evidence and pipeline context")
    e1, e2, e3 = st.columns(3)
    with e1:
        st.markdown(
            f"""<div class="soft-card"><div class="eyebrow">Clinical development</div>
            <p><b>{data.get('registered_trials', data['trials'])}</b> registered studies found</p>
            <p><b>{data['trials']}</b> currently active studies</p>
            <p><b>Phase {data['phase_max']}</b> highest modeled or observed phase</p></div>""",
            unsafe_allow_html=True,
        )
    with e2:
        st.markdown(
            f"""<div class="soft-card"><div class="eyebrow">Research base</div>
            <p><b>{data['pubmed_citations']:,}</b> PubMed title or abstract matches</p>
            <p><b>${data['nih_funding']:,}</b> NIH funding estimate</p>
            <p><b>{data.get('nih_projects', 'Not refreshed')}</b> NIH projects in live query</p></div>""",
            unsafe_allow_html=True,
        )
    with e3:
        fda_url = "https://www.accessdata.fda.gov/scripts/opdlisting/oopd/index.cfm"
        trials_url = f"https://clinicaltrials.gov/search?cond={quote_plus(selected)}"
        pubmed_url = f"https://pubmed.ncbi.nlm.nih.gov/?term={quote_plus(selected)}"
        st.markdown(
            f"""<div class="soft-card"><div class="eyebrow">Verify at source</div>
            <p><a href="{trials_url}" target="_blank">Open ClinicalTrials.gov search</a></p>
            <p><a href="{pubmed_url}" target="_blank">Open PubMed search</a></p>
            <p><a href="{fda_url}" target="_blank">Search FDA orphan designations</a></p></div>""",
            unsafe_allow_html=True,
        )

    records = data.get("trial_records", [])
    if records:
        st.markdown("#### Sample live trial records")
        st.dataframe(pd.DataFrame(records), hide_index=True, use_container_width=True)

with compare_tab:
    c1, c2 = st.columns(2)
    names = sorted(DISEASE_DB)
    with c1:
        a_name = st.selectbox("Disease A", names, index=names.index("Visceral Myopathy"), key="compare_a")
    with c2:
        b_default = "Spinal Muscular Atrophy"
        b_name = st.selectbox("Disease B", names, index=names.index(b_default), key="compare_b")
    a = st.session_state.live_data.get(a_name, DISEASE_DB[a_name])
    b = st.session_state.live_data.get(b_name, DISEASE_DB[b_name])
    ma, mb = st.columns(2)
    ma.metric(a_name, f"{opportunity_score(a)}/10", risk_label(opportunity_score(a))[0])
    mb.metric(b_name, f"{opportunity_score(b)}/10", risk_label(opportunity_score(b))[0])
    st.plotly_chart(compare_chart(a, a_name, b, b_name), use_container_width=True, config={"displayModeBar": False})
    comparison = pd.DataFrame({
        "Metric": ["Biological validation", "Trial feasibility", "Patient population", "Regulatory clarity", "Commercial incentive", "Active trials", "PubMed records", "NIH funding"],
        a_name: [a["bio"], a["feasibility"], a["population"], a["regulatory"], a["commercial"], a["trials"], a["pubmed_citations"], a["nih_funding"]],
        b_name: [b["bio"], b["feasibility"], b["population"], b["regulatory"], b["commercial"], b["trials"], b["pubmed_citations"], b["nih_funding"]],
    })
    st.dataframe(comparison, hide_index=True, use_container_width=True)

with portfolio_tab:
    rows = []
    for name, seed in DISEASE_DB.items():
        d = st.session_state.live_data.get(name, seed)
        score = opportunity_score(d)
        rows.append({
            "Disease": name, "Category": d["category"], "Gene / pathway": d["gene"],
            "Translation score": score, "Risk": risk_label(score)[0], "Prevalence": d["prevalence"],
            "Active trials": d["trials"], "PubMed records": d["pubmed_citations"],
            "NIH funding": d["nih_funding"], "Evidence": d["data_source"],
        })
    portfolio = pd.DataFrame(rows)
    visible = portfolio.copy()
    if category_filter:
        visible = visible[visible["Category"].isin(category_filter)]
    visible = visible[visible["Translation score"] >= min_score].sort_values("Translation score")

    p1, p2, p3, p4 = st.columns(4)
    p1.metric("Diseases in catalog", len(portfolio))
    p2.metric("Visible after filters", len(visible))
    p3.metric("Higher-risk programs", int((visible["Translation score"] < 4.5).sum()))
    p4.metric("Median score", f"{visible['Translation score'].median():.1f}" if len(visible) else "N/A")

    scatter = px.scatter(
        visible, x="Translation score", y="Prevalence", size="Active trials", color="Category",
        hover_name="Disease", log_y=True, size_max=32,
        labels={"Prevalence": "Estimated U.S. patient population", "Translation score": "Translation score (higher is more favorable)"},
    )
    scatter.update_layout(
        paper_bgcolor="rgba(0,0,0,0)", plot_bgcolor="#ffffff", legend_title_text="Category",
        margin=dict(t=25, b=20, l=20, r=20), height=510,
    )
    scatter.update_xaxes(gridcolor="#e6eeea")
    scatter.update_yaxes(gridcolor="#e6eeea")
    st.plotly_chart(scatter, use_container_width=True, config={"displayModeBar": False})

    st.dataframe(visible, hide_index=True, use_container_width=True, height=520)
    st.download_button(
        "Download filtered portfolio CSV",
        data=visible.to_csv(index=False).encode("utf-8"),
        file_name="rdti_orphan_disease_portfolio.csv",
        mime="text/csv",
    )

with methods_tab:
    st.markdown("## How the model works")
    st.markdown(
        """
The platform combines five 1-to-10 dimensions into a weighted translation score:

- **Biological validation, 25%:** strength and specificity of the disease mechanism and therapeutic rationale.
- **Trial feasibility, 25%:** enrollment practicality, endpoint quality, natural-history readiness, and operational burden.
- **Patient population, 15%:** relative ability to identify and recruit an addressable population.
- **Regulatory clarity, 20%:** precedent, surrogate endpoint acceptance, and pathway maturity.
- **Commercial incentive, 15%:** addressable market, competition, development cost, and likely reimbursement support.

The 100-disease catalog starts with modeled baseline values so every screen remains usable without internet access. Pressing **Refresh live evidence** replaces trial counts, literature counts, and recent NIH funding with current API results for the selected disease. Those live fields provide evidence inputs, while the five translation dimensions remain editable model assumptions in this prototype.
"""
    )
    st.markdown("### Data provenance")
    st.markdown(
        """
- Clinical trial records come from the ClinicalTrials.gov API v2.
- Literature counts come from NCBI PubMed E-utilities.
- Federal award estimates come from the NIH RePORTER API v2 and cover the latest five fiscal years included in the request.
- FDA orphan designation records are linked to the official searchable database because FDA does not expose this specific database through a documented public REST endpoint in the same way as the other three sources.
"""
    )
    st.warning("Baseline prevalence, approvals, and dimension scores are prototype estimates. They require source-level validation before publication, investment use, or external scientific claims.")

st.markdown("---")
st.markdown(
    '<div class="small-note">RDTI · Built by Hanyu Su · Research prioritization prototype · Public evidence refreshed only when requested.</div>',
    unsafe_allow_html=True,
)
