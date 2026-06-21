import streamlit as st
import requests
import json
import base64
import re
import tempfile
import uuid
from pathlib import Path
from gtts import gTTS
# ==========================================================
# CONFIGURATION
# ==========================================================

# Recommended: move this key to Streamlit secrets later.
LYZR_API_KEY = "sk-default-91vxULtoe3fbi2CZBHxPsbRyWlxEfbJJ"

LYZR_BASE_URL = "https://agent-prod.studio.lyzr.ai/v3/inference/chat/"

KURAL_AGENT_ID = "6a31472d941c9fc16b30ca03"
PEYCHU_AGENT_ID = "6a32ac505807cd500d835e39"
STORY_AGENT_ID = "6a32aca23e437aa6f62037f8"
SITUATION_AGENT_ID = "6a34005f25c1399e020621b7"

HEADERS = {
    "Content-Type": "application/json",
    "x-api-key": LYZR_API_KEY,
}

# ==========================================================
# HELPER FUNCTIONS
# ==========================================================

def call_agent(agent_id: str, message: str) -> str:
    """Generic Lyzr Agent Call."""
    payload = {
        "user_id": "streamlit-user",
        "agent_id": agent_id,
        # Use a fresh session for every Proceed click so the agent does not reuse old context.
        "session_id": st.session_state.get("agent_session_id", "thirukural-ai"),
        "message": message,
    }

    try:
        response = requests.post(
            LYZR_BASE_URL,
            headers=HEADERS,
            json=payload,
            timeout=120,
        )
        response.raise_for_status()
        data = response.json()

        if "response" in data:
            return data["response"]

        if "message" in data:
            return data["message"]

        return json.dumps(data, indent=2, ensure_ascii=False)

    except Exception as e:
        return f"Error: {str(e)}"


def _extract_first_json_object(text: str) -> str:
    """Return the first balanced JSON object from a text response."""
    text = text.strip()
    start = text.find("{")
    if start == -1:
        raise ValueError("No JSON object found in response")

    in_string = False
    escape = False
    depth = 0

    for i in range(start, len(text)):
        ch = text[i]

        if escape:
            escape = False
            continue

        if ch == "\\":
            escape = True
            continue

        if ch == '"':
            in_string = not in_string
            continue

        if not in_string:
            if ch == "{":
                depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    return text[start : i + 1]

    # Fallback: use first { through last } if a perfectly balanced object was not found.
    end = text.rfind("}")
    if end != -1 and end > start:
        return text[start : end + 1]

    raise ValueError("Could not extract a complete JSON object")


def _escape_control_chars_inside_strings(text: str) -> str:
    """Escape literal newlines/tabs inside JSON strings without touching JSON syntax."""
    result = []
    in_string = False
    escape = False

    for ch in text:
        if escape:
            result.append(ch)
            escape = False
            continue

        if ch == "\\":
            result.append(ch)
            escape = True
            continue

        if ch == '"':
            result.append(ch)
            in_string = not in_string
            continue

        if in_string:
            if ch == "\n":
                result.append("\\n")
            elif ch == "\r":
                result.append("\\r")
            elif ch == "\t":
                result.append("\\t")
            else:
                result.append(ch)
        else:
            result.append(ch)

    return "".join(result)


def _remove_trailing_commas(text: str) -> str:
    return re.sub(r",\s*([}\]])", r"\1", text)



def _repair_known_agent_json_issues(json_text: str) -> str:
    """
    Repair common JSON formatting mistakes from Lyzr agent output.

    The UI does not use meaning_summary. If the agent accidentally returns it,
    remove it before parsing. This avoids failures caused by unescaped quotes
    inside meaning_summary.
    """

    # Remove meaning_summary when it appears before scholar_explanations.
    # Example bad response:
    # "meaning_summary":""Such we are...", "scholar_explanations":[...]
    json_text = re.sub(
        r'\s*,?\s*"meaning_summary"\s*:\s*.*?(?=,\s*"scholar_explanations"\s*:)',
        "",
        json_text,
        flags=re.DOTALL,
    )

    # Remove meaning_summary when it appears before confidence_score.
    json_text = re.sub(
        r'\s*,?\s*"meaning_summary"\s*:\s*.*?(?=,\s*"confidence_score"\s*:)',
        "",
        json_text,
        flags=re.DOTALL,
    )

    # If meaning_summary is valid JSON and at the end, remove it too.
    json_text = re.sub(
        r',\s*"meaning_summary"\s*:\s*"[^"]*"\s*(?=})',
        "",
        json_text,
        flags=re.DOTALL,
    )

    return json_text

def parse_json_response(response_text: str) -> dict:
    """
    Robustly parse JSON returned by Lyzr agents.

    Handles:
    - ```json markdown wrappers
    - extra text before/after JSON
    - literal newlines inside JSON string values
    - trailing commas before } or ]
    """
    if not response_text or not response_text.strip():
        raise ValueError("Empty response received from agent")

    cleaned = response_text.strip()
    cleaned = cleaned.replace("```json", "")
    cleaned = cleaned.replace("```", "")
    # Do not replace Tamil smart quotes “ ” with double quotes here.
    # They are valid characters inside JSON string values and replacing them breaks valid JSON.

    json_text = _extract_first_json_object(cleaned)
    json_text = _repair_known_agent_json_issues(json_text)
    json_text = _escape_control_chars_inside_strings(json_text)
    json_text = _remove_trailing_commas(json_text)

    try:
        return json.loads(json_text, strict=False)
    except json.JSONDecodeError:
        # Last-resort cleanup for some agent outputs that contain raw line breaks
        # between JSON tokens or stray semicolons.
        json_text = re.sub(r";\s*([}\]])", r"\1", json_text)
        json_text = _remove_trailing_commas(json_text)
        return json.loads(json_text, strict=False)

def create_audio_base64(text: str) -> str:
    """Create Tamil MP3 audio using gTTS and return base64 string."""
    if not text.strip():
        return ""

    tts = gTTS(text=text, lang="ta")
    with tempfile.NamedTemporaryFile(delete=False, suffix=".mp3") as fp:
        tts.save(fp.name)
        with open(fp.name, "rb") as audio_file:
            audio_bytes = audio_file.read()

    return base64.b64encode(audio_bytes).decode()


def extract_scholar_explanations(scholar_explanations: list) -> tuple[str, str, str]:
    """
    Extract the three Tamil scholar explanations.

    Supports both short keys from the old agent:
    - mv, sp, mk

    and Tamil names from newer agent output:
    - மு. வரதராசனார் / மு.வரதராசனார்
    - சா. பாப்பையா / சா.பாப்பையா
    - மு. கலைஞர் / மு.கலைஞர் / கலைஞர்
    """
    mv_explanation = ""
    sp_explanation = ""
    mk_explanation = ""

    for scholar in scholar_explanations:
        scholar_name = str(scholar.get("scholar", "")).strip()
        explanation = scholar.get("explanation", "")

        normalized = scholar_name.replace(" ", "").replace(".", "").lower()

        if normalized in ["mv", "முவரதராசனார்", "வரதராசனார்"]:
            mv_explanation = explanation
        elif normalized in ["sp", "சாபாப்பையா", "பாப்பையா"]:
            sp_explanation = explanation
        elif normalized in ["mk", "முகலைஞர்", "கலைஞர்"]:
            mk_explanation = explanation

    return mv_explanation, sp_explanation, mk_explanation




def init_session_state() -> None:
    defaults = {
        "show_results": False,
        "kural_response": "",
        "kural_number": "",
        "kural_line1": "",
        "kural_line2": "",
        "kural_athikaram": "",
        "kural_iyal": "",
        "kural_paal": "",
        "mv_explanation": "",
        "sp_explanation": "",
        "mk_explanation": "",
        "kids_explanation": "",
        "teen_explanation": "",
        "adult_explanation": "",
        "story_prompt": "",
        "story_response": "",
        "audio_base64": "",
    }

    for key, value in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = value


def save_results_to_session(
    *,
    kural_response: str,
    kural_number: str,
    kural_line1: str,
    kural_line2: str,
    kural_athikaram: str,
    kural_iyal: str,
    kural_paal: str,
    mv_explanation: str,
    sp_explanation: str,
    mk_explanation: str,
    kids_explanation: str,
    teen_explanation: str,
    adult_explanation: str,
    story_prompt: str,
    story_response: str,
    audio_base64: str,
) -> None:
    st.session_state.kural_response = kural_response
    st.session_state.kural_number = kural_number
    st.session_state.kural_line1 = kural_line1
    st.session_state.kural_line2 = kural_line2
    st.session_state.kural_athikaram = kural_athikaram
    st.session_state.kural_iyal = kural_iyal
    st.session_state.kural_paal = kural_paal

    st.session_state.mv_explanation = mv_explanation
    st.session_state.sp_explanation = sp_explanation
    st.session_state.mk_explanation = mk_explanation

    st.session_state.kids_explanation = kids_explanation
    st.session_state.teen_explanation = teen_explanation
    st.session_state.adult_explanation = adult_explanation

    st.session_state.story_prompt = story_prompt
    st.session_state.story_response = story_response
    st.session_state.audio_base64 = audio_base64
    st.session_state.show_results = True


def render_kural_box() -> None:
    """Render Kural card with audio icon in the top-right corner."""
    audio_base64 = st.session_state.audio_base64
    kural_line1 = st.session_state.kural_line1
    kural_line2 = st.session_state.kural_line2

    st.components.v1.html(
        f"""
        <style>
        .kural-box {{
            position: relative;
            background:#fdf6d8;
            border-radius:15px;
            padding:35px 55px 35px 55px;
            border:2px solid #d4b96f;
            font-size:28px;
            color:#3e2723;
            text-align:center;
            line-height:1.8;
            min-height:120px;
            box-sizing:border-box;
        }}

        .speaker-btn {{
            position:absolute;
            top:12px;
            right:15px;
            background:#ffffffcc;
            border:1px solid #d4b96f;
            border-radius:50%;
            width:42px;
            height:42px;
            font-size:22px;
            cursor:pointer;
            display:flex;
            align-items:center;
            justify-content:center;
        }}

        .speaker-btn:hover {{
            background:#fff7cc;
        }}
        </style>

        <audio id="kuralAudio">
            <source src="data:audio/mp3;base64,{audio_base64}" type="audio/mp3">
        </audio>

        <div class="kural-box">
            <button class="speaker-btn"
                    title="Read Kural"
                    onclick="document.getElementById('kuralAudio').play()">
                🔊
            </button>

            {kural_line1}<br>
            {kural_line2}
        </div>
        """,
        height=190,
    )


def render_agent_workflow(
    step1_status: str = "waiting",
    step2_status: str = "waiting",
    step3_status: str = "waiting",
    step1_desc: str = "Waiting for Kural search...",
    step2_desc: str = "Waiting for Kural details...",
    step3_desc: str = "Waiting for explanation...",
) -> str:
    """Return HTML for the 3-agent vertical workflow."""

    status_icon = {
        "done": "✓",
        "running": "↻",
        "waiting": "⌛",
        "failed": "!",
    }

    status_class = {
        "done": "done",
        "running": "running",
        "waiting": "waiting",
        "failed": "failed",
    }

    def row(status: str, icon: str, title: str, desc: str) -> str:
        status_css = status_class.get(status, "waiting")
        symbol = status_icon.get(status, "⌛")

        return f"""<div class="agent-row">
  <div class="agent-status {status_css}">{symbol}</div>
  <div class="agent-mini-card">
    <div class="agent-title-line">{icon} {title}</div>
    <div class="agent-desc-line">{desc}</div>
  </div>
</div>"""

    return f"""<div class="workflow-title">⚙️ Agent Workflow</div>
<div class="agent-flow">
{row(step1_status, "🔍", "Kural Retrieval Agent", step1_desc)}
{row(step2_status, "💬", "Peychu Explanation Agent", step2_desc)}
{row(step3_status, "✍️", "Story Generation Agent", step3_desc)}
</div>"""


def _html_escape(text: str) -> str:
    """Basic HTML escaping for generated flash card text."""
    return (
        str(text or "")
        .replace("&", "&amp;")
        .replace("<", "&lt;")
        .replace(">", "&gt;")
        .replace('"', "&quot;")
        .replace("'", "&#39;")
    )


def render_flashcard_print_button() -> None:
    """
    Render a print/save-as-PDF flashcard button.

    v7.5:
    - Opens a clean print document.
    - Page 1 contains ONLY the Kural front side.
    - Page 2 contains ONLY the explanation back side.
    - Unicode-safe Tamil rendering using json.dumps, not base64/atob.
    """
    if not st.session_state.get("show_results", False):
        return

    kural_number = _html_escape(st.session_state.kural_number)
    athikaram = _html_escape(st.session_state.kural_athikaram)
    line1 = _html_escape(st.session_state.kural_line1)
    line2 = _html_escape(st.session_state.kural_line2)
    mv_explanation = _html_escape(st.session_state.mv_explanation)
    teen_explanation = _html_escape(st.session_state.teen_explanation)

    flashcard_html = f"""
<!DOCTYPE html>
<html lang="ta">
<head>
<meta charset="UTF-8">
<title>LiteraryAI Bridge Flash Card</title>

<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+Tamil:wght@400;500;600;700;800&display=swap" rel="stylesheet">

<style>
    * {{
        box-sizing: border-box;
    }}

    html, body {{
        margin: 0;
        padding: 0;
        background: #fff8e7;
        color: #1f2937;
        font-family: "Noto Sans Tamil", "Nirmala UI", "Latha", Arial, sans-serif;
        -webkit-print-color-adjust: exact;
        print-color-adjust: exact;
    }}

    body {{
        padding: 24px;
    }}

    .print-page {{
        width: 100%;
        min-height: calc(100vh - 48px);
        background: #fff8e7;
        display: flex;
        align-items: stretch;
        justify-content: center;
    }}

    .page-break {{
        break-after: page;
        page-break-after: always;
        height: 0;
        margin: 0;
        padding: 0;
    }}

    .flashcard-side {{
        width: 100%;
        border: 4px solid #b8860b;
        border-radius: 30px;
        background: #fffdf5;
        padding: 34px;
        overflow: hidden;
    }}

    .flashcard-top {{
        display: flex;
        justify-content: space-between;
        align-items: flex-start;
        gap: 16px;
        font-size: 20px;
        color: #3e2723;
        margin-bottom: 28px;
        font-weight: 700;
    }}

    .flashcard-top div:last-child {{
        text-align: right;
    }}

    .flashcard-title {{
        text-align: center;
        font-size: 38px;
        font-weight: 800;
        color: #3e2723;
        margin-bottom: 34px;
    }}

    .kural-frame {{
        border: 3px solid #b8860b;
        border-radius: 26px;
        padding: 54px 34px;
        text-align: center;
        font-size: 38px;
        line-height: 2;
        min-height: 3.8in;
        display: flex;
        align-items: center;
        justify-content: center;
        margin: 32px 36px 0 36px;
    }}

    .flashcard-footer {{
        text-align: center;
        margin-top: 34px;
        font-size: 15px;
        color: #6b7280;
    }}

    .back-side {{
        padding: 38px 48px;
    }}

    .explanation-box {{
        border-radius: 24px;
        padding: 28px;
        margin-bottom: 34px;
        line-height: 1.85;
        font-size: 21px;
        overflow: hidden;
    }}

    .mv-box {{
        background: #eef6ff;
        border: 3px solid #60a5fa;
        min-height: 2.25in;
    }}

    .teen-box {{
        background: #f0fdf4;
        border: 3px solid #22c55e;
        min-height: 2.75in;
    }}

    .explanation-title {{
        font-size: 24px;
        font-weight: 800;
        color: #3e2723;
        margin-bottom: 14px;
    }}

    @media print {{
        @page {{
            size: landscape;
            margin: 0.35in;
        }}

        html, body {{
            width: 100%;
            height: auto;
            background: #fff8e7 !important;
        }}

        body {{
            padding: 0 !important;
        }}

        .print-page {{
            min-height: 7.0in;
            height: 7.0in;
            page-break-inside: avoid;
            break-inside: avoid;
        }}

        .page-break {{
            display: block;
            page-break-after: always;
            break-after: page;
        }}

        .flashcard-side {{
            page-break-inside: avoid;
            break-inside: avoid;
        }}
    }}
</style>
</head>

<body>

    <!-- PAGE 1: FRONT -->
    <div class="print-page">
        <section class="flashcard-side front-side">
            <div class="flashcard-top">
                <div>Kural No: {kural_number}</div>
                <div>அதிகாரம்: {athikaram}</div>
            </div>

            <div class="flashcard-title">திருக்குறள்</div>

            <div class="kural-frame">
                <div>
                    {line1}<br>
                    {line2}
                </div>
            </div>

            <div class="flashcard-footer">
                LiteraryAI Bridge - Flash Card
            </div>
        </section>
    </div>

    <div class="page-break"></div>

    <!-- PAGE 2: BACK -->
    <div class="print-page">
        <section class="flashcard-side back-side">
            <div class="flashcard-title">விளக்கம்</div>

            <div class="explanation-box mv-box">
                <div class="explanation-title">மு. வரதராசனார் விளக்கம்</div>
                <div>{mv_explanation}</div>
            </div>

            <div class="explanation-box teen-box">
                <div class="explanation-title">இளையோர் விளக்கம்</div>
                <div>{teen_explanation}</div>
            </div>
        </section>
    </div>

</body>
</html>
"""

    safe_html = json.dumps(flashcard_html, ensure_ascii=False)

    st.components.v1.html(
        f"""
        <button onclick='
            const printWindow = window.open("", "_blank");
            printWindow.document.open();
            printWindow.document.write({safe_html});
            printWindow.document.close();
            printWindow.focus();
            setTimeout(() => printWindow.print(), 900);
        '
            style="
                width:100%;
                background:#2563eb;
                color:white;
                border:none;
                padding:10px 12px;
                border-radius:10px;
                cursor:pointer;
                font-size:15px;
                font-weight:600;">
            🃏 Generate Flash Card
        </button>
        """,
        height=46,
    )




def render_results() -> None:
    """Render Insights from session_state. This is intentionally outside process_btn."""
    if not st.session_state.get("show_results", False):
        return

    st.divider()

    st.markdown(
        '<div id="bridge-insights"></div>',
        unsafe_allow_html=True,
    )

    insight_col, flash_col = st.columns([7, 2])

    with insight_col:
        st.header("📚 Bridge Insights")

    with flash_col:
        st.write("")
        render_flashcard_print_button()

    # ------------------------------------------------------
    # KURAL DATA
    # ------------------------------------------------------
    with st.container(border=True):
        st.subheader("📖 Thirukkural")

        col1, col2, col3, col4 = st.columns(4)

        with col1:
            st.info(f"குறள் எண் : {st.session_state.kural_number}")
        with col2:
            st.info(f"அதிகாரம் : {st.session_state.kural_athikaram}")
        with col3:
            st.info(f"இயல் : {st.session_state.kural_iyal}")
        with col4:
            st.info(f"பால் : {st.session_state.kural_paal}")

        render_kural_box()

    st.write("")

    # ------------------------------------------------------
    # SCHOLAR EXPLANATIONS
    # ------------------------------------------------------
    with st.container(border=True):
        st.subheader("✒️ Scholar Explanations")

        scholar_tabs = st.tabs([
            "மு.வரதராசனார்",
            "சா.பாப்பையா",
            "மு.கலைஞர்",
        ])

        with scholar_tabs[0]:
            st.write(st.session_state.mv_explanation)

        with scholar_tabs[1]:
            st.write(st.session_state.sp_explanation)

        with scholar_tabs[2]:
            st.write(st.session_state.mk_explanation)

    st.write("")

    # ------------------------------------------------------
    # PEYCHU TAMIL
    # ------------------------------------------------------
    with st.container(border=True):
        st.subheader("💬 பேச்சுத் தமிழ் விளக்கம்")

        peychu_tabs = st.tabs([
            "குழந்தைகள்",
            "இளையோர்",
            "பெரியோர்",
        ])

        with peychu_tabs[0]:
            st.success(st.session_state.kids_explanation)

        with peychu_tabs[1]:
            st.success(st.session_state.teen_explanation)

        with peychu_tabs[2]:
            st.success(st.session_state.adult_explanation)

    st.write("")

    # ------------------------------------------------------
    # STORY WITH REGENERATE BUTTON
    # ------------------------------------------------------
    with st.container(border=True):
        col1, col2 = st.columns([8, 1])

        with col1:
            st.subheader("🎨 வாழ்க்கைக் கதை / உதாரணம்")

        with col2:
            if st.button("🔄", key="regen_story", help="Regenerate story only"):
                with st.spinner("Generating new story..."):
                    regenerate_prompt = f"""
                    Generate a completely NEW and UNIQUE story.
                    Do not repeat the previous story.
                    Use different characters, setting, and events.
                    Regeneration ID: {uuid.uuid4()}

                    Base Story Prompt:
                    {st.session_state.story_prompt}
                    """

                    st.session_state.story_response = call_agent(
                        STORY_AGENT_ID,
                        regenerate_prompt,
                    )
                st.rerun()

        st.markdown(st.session_state.story_response)


# ==========================================================
# PAGE CONFIG
# ==========================================================

st.set_page_config(
    page_title="LiteraryAI Bridge",
    page_icon="🧘",
    layout="wide",
)
st.markdown("""
<style>

.block-container {
    padding-top: 0rem;
    padding-bottom: 1rem;
}

header[data-testid="stHeader"]{
    display:none;
}

#MainMenu{
    visibility:hidden;
}

footer{
    visibility:hidden;
}

</style>
""", unsafe_allow_html=True)
init_session_state()

# ==========================================================
# CUSTOM CSS
# ==========================================================

st.markdown(
    """
    <style>
    .main {
        background-color:#f8fafc;
    }

    .big-title{
        font-size:30px;
        font-weight:700;
        color:#1e1b4b;
    }

    .sub-title{
        color:#64748b;
        font-size:14px;
    }

    .section-card{
        border:1px solid #e2e8f0;
        border-radius:15px;
        padding:20px;
        background:white;
    }

    .meta-box{
        padding:8px 15px;
        background:#f5f3ff;
        border-radius:10px;
        color:#4338ca;
        font-weight:600;
    }

    .input-card {
        border:1px solid #e5e7eb;
        border-radius:18px;
        padding:20px;
        background:#ffffff;
        box-shadow:0 2px 8px rgba(15,23,42,0.05);
        min-height:285px;
    }

    .workflow-card {
        border:1px solid #e5e7eb;
        border-radius:18px;
        padding:20px;
        background:#ffffff;
        box-shadow:0 2px 8px rgba(15,23,42,0.05);
        min-height:285px;
    }

    .wizard-title {
        font-size:24px;
        font-weight:700;
        color:#111827;
        margin-bottom:14px;
    }

    .workflow-title {
        font-size:24px;
        font-weight:700;
        color:#111827;
        margin-bottom:14px;
    }

    .agent-flow {
        position:relative;
        padding:4px 0 0 0;
    }

    .agent-flow::before {
        content:"";
        position:absolute;
        left:18px;
        top:54px;
        bottom:36px;
        width:3px;
        background:#d1fae5;
        border-radius:3px;
    }

    .agent-row {
        position:relative;
        display:flex;
        align-items:flex-start;
        gap:12px;
        margin:14px 0;
    }

    .agent-status {
        width:38px;
        height:38px;
        border-radius:50%;
        display:flex;
        align-items:center;
        justify-content:center;
        font-size:20px;
        z-index:2;
        background:#f3f4f6;
        border:2px solid #e5e7eb;
        flex-shrink:0;
    }

    .agent-status.done {
        background:#22c55e;
        color:white;
        border-color:#22c55e;
    }

    .agent-status.running {
        background:#eef2ff;
        color:#4f46e5;
        border-color:#818cf8;
        animation:pulse 1.2s infinite;
    }

    .agent-status.waiting {
        background:#f9fafb;
        color:#9ca3af;
    }

    .agent-status.failed {
        background:#fee2e2;
        color:#dc2626;
        border-color:#fca5a5;
    }

    @keyframes pulse {
        0% { box-shadow:0 0 0 0 rgba(79,70,229,0.35); }
        70% { box-shadow:0 0 0 8px rgba(79,70,229,0); }
        100% { box-shadow:0 0 0 0 rgba(79,70,229,0); }
    }

    .agent-mini-card {
        flex:1;
        background:#ffffff;
        border:1px solid #e5e7eb;
        border-radius:14px;
        padding:12px;
        box-shadow:0 2px 8px rgba(15,23,42,0.04);
    }

    .agent-title-line {
        font-size:15px;
        font-weight:700;
        color:#111827;
        margin-bottom:4px;
    }

    .agent-desc-line {
        font-size:13px;
        color:#6b7280;
        line-height:1.35;
    }

    /* Compact the top input workflow area */
    div[data-testid="stTextArea"] textarea {
        min-height:95px !important;
    }

    div[data-testid="stVerticalBlockBorderWrapper"] {
        border-radius:14px;
        box-shadow:0 2px 10px rgba(15,23,42,0.06);
    }
    max-height:220px;
    object-fit:cover;
    border-radius:20px;
    </style>
    """,
    unsafe_allow_html=True,
)

# ==========================================================
# HEADER
# ==========================================================

# st.markdown(
    # """
    # <div class='big-title'>🧘 LiteraryAI Bridge - Thirukkural</div>
    # <div class='sub-title'>Ancient Wisdom for Modern Life</div>
    # """,
    # unsafe_allow_html=True,
# )

st.markdown(
"""
<style>
.banner-img{
    width:100%;
    max-height:200px;
    object-fit:cover;
    border-radius:20px;
    overflow:hidden;
}
</style>
""",
unsafe_allow_html=True
)

with open("assets/banner.png", "rb") as f:
    data = f.read()

encoded = base64.b64encode(data).decode()

st.markdown(
f"""
<img class="banner-img"
src="data:image/png;base64,{encoded}">
""",
unsafe_allow_html=True
)

# st.divider()

# ==========================================================
# INPUT SECTION
# ==========================================================

layout_col, workflow_col = st.columns([2.55, 1], gap="medium", vertical_alignment="top")

with layout_col:
    with st.container(border=True):
        st.markdown('<div class="wizard-title">🔍 Search & Situation Wizard</div>', unsafe_allow_html=True)

        search_col, situation_col = st.columns(2, gap="medium")

        with search_col:
            search_input = st.text_area(
                "Search Criterion",
                height=95,
                placeholder="Kural number, starting word, ending word or full kural...",
            )

        with situation_col:
            situation_input = st.text_area(
                "Explain your Situation",
                height=95,
                placeholder="Describe your situation...",
            )

        process_btn = st.button(
            "Proceed to Interpretation",
            use_container_width=True,
            type="primary",
        )

        # Spacer to visually align this search card height with the Agent Workflow card.
        # Adjust this height if you later change the workflow card content.
        st.markdown("<div style='height:78px;'></div>", unsafe_allow_html=True)

with workflow_col:
    with st.container(border=True):
        workflow_box = st.empty()
        workflow_box.markdown(
            render_agent_workflow(),
            unsafe_allow_html=True,
        )

# ==========================================================
# MAIN PROCESSING - ONLY GENERATION HAPPENS HERE
# ==========================================================

if process_btn:
    # Fresh Lyzr session for every new search/Kural number.
    # This prevents old agent context from mixing with the new response.
    st.session_state.agent_session_id = f"thirukural-ai-{uuid.uuid4()}"

    if not search_input.strip() and not situation_input.strip():
        st.warning("Please enter either a Kural search input or a situation")
        workflow_box.markdown(
            render_agent_workflow(
                step1_status="waiting",
                step1_desc="Please enter a Kural search or situation.",
            ),
            unsafe_allow_html=True,
        )
        st.stop()

    # ------------------------------------------------------
    # STEP 1 - KURAL IDENTIFICATION
    # ------------------------------------------------------
    workflow_box.markdown(
        render_agent_workflow(
            step1_status="running",
            step1_desc="Searching Thirukkural knowledge base...",
            step2_desc="Waiting for Kural details...",
            step3_desc="Waiting for explanation...",
        ),
        unsafe_allow_html=True,
    )

    if search_input.strip():
        kural_response = call_agent(KURAL_AGENT_ID, search_input.strip())
    else:
        situation_response = call_agent(
            SITUATION_AGENT_ID,
            situation_input
        )

        situation_data = parse_json_response(situation_response)

        kural_number = situation_data["kural_number"]

        # Use Kural Agent as source of truth
        kural_response = call_agent(
            KURAL_AGENT_ID,
            str(kural_number)
        )
        # kural_response = call_agent(SITUATION_AGENT_ID, situation_input.strip())

    # st.write("Situation Input:", situation_input)
    # st.code(kural_response)
    
    try:
        kural_data = parse_json_response(kural_response)
    except Exception as e:
        workflow_box.markdown(
            render_agent_workflow(
                step1_status="failed",
                step1_desc="Kural Agent failed. Check JSON response below.",
                step2_desc="Not started.",
                step3_desc="Not started.",
            ),
            unsafe_allow_html=True,
        )
        st.error("Could not parse Kural Agent JSON response.")
        st.code(kural_response)
        st.exception(e)
        st.stop()

    kural_number = kural_data.get("kural_number", "")
    kural_line1 = kural_data.get("Line1", "")
    kural_line2 = kural_data.get("Line2", "")
    kural_iyal = kural_data.get("iyal", "")
    kural_athikaram = kural_data.get("athikaram", "")
    kural_paal = kural_data.get("paal", "")

    kural_text = f"{kural_line1} {kural_line2}"
    audio_base64 = create_audio_base64(kural_text)

    scholar_explanations = kural_data.get("scholar_explanations", [])
    mv_explanation, sp_explanation, mk_explanation = extract_scholar_explanations(
        scholar_explanations
    )

    # ------------------------------------------------------
    # STEP 2 - PEYCHU TAMIL EXPLANATIONS
    # ------------------------------------------------------
    workflow_box.markdown(
        render_agent_workflow(
            step1_status="done",
            step2_status="running",
            step1_desc="Best matching Kural identified.",
            step2_desc="Generating conversational Tamil explanation...",
            step3_desc="Waiting for explanation...",
        ),
        unsafe_allow_html=True,
    )

    peychu_response = call_agent(PEYCHU_AGENT_ID, kural_response)

    try:
        peychu_data = parse_json_response(peychu_response)
    except Exception as e:
        workflow_box.markdown(
            render_agent_workflow(
                step1_status="done",
                step2_status="failed",
                step1_desc="Best matching Kural identified.",
                step2_desc="Peychu Agent failed. Check JSON response below.",
                step3_desc="Not started.",
            ),
            unsafe_allow_html=True,
        )
        st.error("Could not parse Peychu Agent JSON response.")
        st.code(peychu_response)
        st.exception(e)
        st.stop()

    kids_explanation = peychu_data.get("kids_explanation", "")
    teen_explanation = peychu_data.get("teen_explanation", "")
    adult_explanation = peychu_data.get("adult_explanation", "")

    # ------------------------------------------------------
    # STEP 3 - STORY GENERATION
    # ------------------------------------------------------
    workflow_box.markdown(
        render_agent_workflow(
            step1_status="done",
            step2_status="done",
            step3_status="running",
            step1_desc="Best matching Kural identified.",
            step2_desc="Conversational Tamil explanation ready.",
            step3_desc="Creating modern example story...",
        ),
        unsafe_allow_html=True,
    )

    story_prompt = f"""
    Generate a NEW and UNIQUE story.
    Use different characters, setting and events.
    Request ID: {uuid.uuid4()}

    Kural Details:
    {kural_response}

    Situation:
    {situation_input}
    """

    story_response = call_agent(STORY_AGENT_ID, story_prompt)

    workflow_box.markdown(
        render_agent_workflow(
            step1_status="done",
            step2_status="done",
            step3_status="done",
            step1_desc="Best matching Kural identified.",
            step2_desc="Conversational Tamil explanation ready.",
            step3_desc="Modern example story ready.",
        ),
        unsafe_allow_html=True,
    )

    save_results_to_session(
        kural_response=kural_response,
        kural_number=kural_number,
        kural_line1=kural_line1,
        kural_line2=kural_line2,
        kural_athikaram=kural_athikaram,
        kural_iyal=kural_iyal,
        kural_paal=kural_paal,
        mv_explanation=mv_explanation,
        sp_explanation=sp_explanation,
        mk_explanation=mk_explanation,
        kids_explanation=kids_explanation,
        teen_explanation=teen_explanation,
        adult_explanation=adult_explanation,
        story_prompt=story_prompt,
        story_response=story_response,
        audio_base64=audio_base64,
    )

    # st.success("Insights Generated Successfully")

    st.components.v1.html(
        """
        <script>
            function scrollToBridgeInsights() {
                const el = window.parent.document.getElementById('bridge-insights');
                if (el) {
                    el.scrollIntoView({
                        behavior: 'smooth',
                        block: 'start'
                    });
                }
            }

            // Give Streamlit enough time to render the results section
            setTimeout(scrollToBridgeInsights, 1000);
        </script>
        """,
        height=0,
    )

# ==========================================================
# OUTPUT SECTION - ALWAYS OUTSIDE process_btn
# ==========================================================

render_results()

# ==========================================================
# FOOTER
# ==========================================================

st.divider()

# st.markdown(
    # """
    # <center>

    # ### கற்க கசடறக் கற்பவை கற்றபின் 
    # ### நிற்க அதற்குத் தக.

    # Learn thoroughly what is to be learned, and then stand by what you have learned.

    # — Thiruvalluvar

    # </center>
    # """,
    # unsafe_allow_html=True,
# )

col1, col2 = st.columns(2)

with col1:
    with st.expander("ℹ️ About App", expanded=True):
        st.markdown("""
**LiteraryAI Bridge** connects the timeless wisdom of **Thirukkural** with modern life through AI-powered explanations, stories, and practical guidance.

✨ Features:

- 📖 Thirukkural retrieval
- 💬 Conversational Tamil explanations
- 👦👧 Kids, Teen & Adult interpretations
- 🎨 Modern example stories
- 🌉 Bridging ancient wisdom with everyday life
""")

with col2:
    with st.expander("👩‍💻 Created By", expanded=True):
        st.markdown("""
### Vidhya Sivachandran

**Powered by Lyzr AI**

📖 Thiruvalluvar's timeless wisdom reimagined through AI.

Special thanks to:

- 🧘 Thiruvalluvar
- 🤖 Lyzr AI Agent Framework
""")