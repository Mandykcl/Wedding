
import streamlit as st
import pandas as pd
import json
from pathlib import Path
from datetime import date, datetime
import plotly.express as px
import base64, mimetypes
from functools import lru_cache

# ---- App identity ----
APP_TITLE = "Our Wedding Planner"

DATA_DIR = Path("data")
ASSETS = Path("assets")
DATA_DIR.mkdir(exist_ok=True)
ASSETS.mkdir(exist_ok=True)

# ---- Simple password gate ----
def _check_password():
    """Return True if the user entered the correct password, else False (and show form)."""
    correct = st.secrets.get("APP_PASSWORD", None)

    if "pw_ok" not in st.session_state:
        st.session_state.pw_ok = False

    if st.session_state.pw_ok:
        return True

    st.title(APP_TITLE)
    with st.form("login"):
        st.write("🔒 Enter password to view")
        pw = st.text_input("Password", type="password")
        remember = st.checkbox("Remember me on this device")
        submit = st.form_submit_button("Enter")

    if submit:
        if correct and pw == correct:
            st.session_state.pw_ok = True if remember else "temp"
            return True
        else:
            st.error("Incorrect password.")
    return False

DEFAULT_SETTINGS = {
    "couple_names": "Mandy & Ashkan",  # <- updated default
    "town_hall_date": None,
    "aghd_date": None,
    "celebration_date": None,
    "town_hall_location": "",
    "aghd_location": "",
    "celebration_location": "",
    "currency": "£"
}

# ---------- File helpers ----------
def _csv_path(name: str) -> Path:
    return DATA_DIR / f"{name}.csv"

def load_settings():
    p = DATA_DIR / "settings.json"
    if p.exists():
        try:
            return json.loads(p.read_text(encoding="utf-8"))
        except Exception:
            pass
    return DEFAULT_SETTINGS.copy()

def save_settings(settings: dict):
    p = DATA_DIR / "settings.json"
    p.write_text(json.dumps(settings, indent=2), encoding="utf-8")

def load_table(name: str, columns: list[str]) -> pd.DataFrame:
    p = _csv_path(name)
    if p.exists():
        try:
            df = pd.read_csv(p)
            for c in columns:
                if c not in df.columns:
                    df[c] = None
            return df[columns] if columns else df
        except Exception:
            pass
    df = pd.DataFrame(columns=columns)
    df.to_csv(p, index=False)
    return df

def save_table(name: str, df: pd.DataFrame):
    df.to_csv(_csv_path(name), index=False)

def currency_symbol():
    return st.session_state.get("settings", {}).get("currency", "£")

# ---------- Background helpers ----------
@lru_cache(maxsize=32)
def _b64(path: str) -> str:
    with open(path, "rb") as f:
        return base64.b64encode(f.read()).decode()

def set_background(image_file: str, darken: float = 0.20):
    """Set page background from assets/ with optional dark overlay for readability."""
    p = ASSETS / image_file if not image_file.startswith(("assets/", "/")) else Path(image_file)
    if not p.exists():
        return  # Skip silently if file missing
    mime = mimetypes.guess_type(p.name)[0] or "image/jpeg"
    encoded = _b64(str(p))
    gradient = f"linear-gradient(rgba(0,0,0,{darken}), rgba(0,0,0,{darken}))"
    css = f"""
    <style>
      .stApp {{
        background: {gradient}, url("data:{mime};base64,{encoded}") center / cover fixed no-repeat !important;
      }}
    </style>
    """
    st.markdown(css, unsafe_allow_html=True)

# Map each tab to a background file name (put your own JPGs in assets/ with these names)
BG = {
    "Dashboard": "bg_dashboard.jpg",
    "Kanban": "bg_kanban.jpg",
    "Tasks": "bg_tasks.jpg",
    "Documents": "bg_documents.jpg",
    "Timeline": "bg_timeline.jpg",
    "Budget": "bg_budget.jpg",
    "Guests": "bg_guests.jpg",
    "Venues & Contacts": "bg_venues.jpg",
    "Moodboard": "bg_moodboard.jpg",
    "Export": "bg_export.jpg",
}

# ---------- Defaults ----------
DEFAULT_DOCS = [
    {"category": "Town Hall", "item": "Passports or valid photo ID", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Town Hall", "item": "Proof of address (council tax/utility bill)", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Town Hall", "item": "Notice of marriage appointment booked", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Town Hall", "item": "Two witnesses arranged (over 16)", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Aghd", "item": "Iranian birth certificates (Shenasnameh) — originals", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Aghd", "item": "National ID cards (if applicable)", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Aghd", "item": "Medical test results per Embassy guidance", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Aghd", "item": "Photos & forms required by the Embassy/Islamic Centre", "owner": "Both", "status": "Not started", "notes": ""},
    {"category": "Aghd", "item": "Witnesses for Aghd (if required)", "owner": "Both", "status": "Not started", "notes": ""},
]

DEFAULT_TASKS = [
    {"milestone": "Town Hall", "task": "Choose and book the Town Hall room", "owner": "Both", "start": "", "due": "", "status": "Not started", "priority":"High", "notes": ""},
    {"milestone": "Town Hall", "task": "Give notice of marriage (≥28 days before)", "owner": "Both", "start": "", "due": "", "status": "Not started", "priority":"High", "notes": ""},
    {"milestone": "Aghd", "task": "Confirm Aghd document list with Islamic Centre", "owner": "Both", "start": "", "due": "", "status": "Not started", "priority":"High", "notes": ""},
    {"milestone": "Aghd", "task": "Book Aghd date and officiant (Aghd-khan)", "owner": "Both", "start": "", "due": "", "status": "Not started", "priority":"Medium", "notes": ""},
    {"milestone": "Embassy", "task": "Legalise test results and submit documents", "owner": "Both", "start": "", "due": "", "status": "Not started", "priority":"Medium", "notes": ""},
    {"milestone": "Celebration", "task": "Shortlist reception venues and photographers", "owner": "Both", "start": "", "due": "", "status": "Not started", "priority":"Low", "notes": ""},
]

# ---------- Date helpers ----------
def _to_date(s):
    if not s:
        return None
    if isinstance(s, date):
        return s
    try:
        return date.fromisoformat(str(s))
    except Exception:
        return None

def _ensure_string_cols(df: pd.DataFrame, cols):
    for c in cols:
        if c in df.columns:
            df[c] = df[c].astype("string").fillna("")
    return df

def _ensure_date_cols(df: pd.DataFrame, cols):
    for c in cols:
        if c in df.columns:
            df[c] = pd.to_datetime(df[c], errors="coerce")
    return df

def _iso_date_series(s: pd.Series) -> pd.Series:
    s = pd.to_datetime(s, errors="coerce")
    return s.dt.date.astype("string")

# ---------- UI sections ----------
def header(settings):
    st.title(APP_TITLE)

    # --- Top-of-page countdown metrics ---
    th_date = _to_date(settings.get("town_hall_date"))
    aghd_date = _to_date(settings.get("aghd_date"))
    cele_date = _to_date(settings.get("celebration_date"))

    def days_to(d):
        if not d: 
            return None
        diff = (d - date.today()).days
        return diff if diff > 0 else 0  # clamp to zero

    cols = st.columns(3)
    with cols[0]:
        if th_date:
            st.metric("Days to Town Hall", days_to(th_date))
        else:
            st.caption("Set Town Hall date in the sidebar")
    with cols[1]:
        if aghd_date:
            st.metric("Days to Aghd", days_to(aghd_date))
        else:
            st.caption("Set Aghd date in the sidebar")
    with cols[2]:
        if cele_date:
            st.metric("Days to Celebration", days_to(cele_date))
        else:
            st.caption("Set Celebration date in the sidebar")

    # --- Sidebar ---
    with st.sidebar:
        st.subheader("Key Details")

        # Fixed couple display (non-editable)
        st.markdown("**Couple:** Mandy & Ashkan")

        th_date_in = st.date_input("Town Hall date", value=th_date)
        aghd_date_in = st.date_input("Aghd date", value=aghd_date)
        cele_date_in = st.date_input("Celebration date", value=cele_date)
        th_loc = st.text_input("Town Hall location", value=settings.get("town_hall_location", ""))
        aghd_loc = st.text_input("Aghd location", value=settings.get("aghd_location", ""))
        cele_loc = st.text_input("Celebration location", value=settings.get("celebration_location", ""))
        curr = st.selectbox("Currency", ["£", "€", "$"], index=["£","€","$"].index(settings.get("currency","£")))

        if st.button("Save details", use_container_width=True):
            new_settings = {
                "couple_names": "Mandy & Ashkan",
                "town_hall_date": th_date_in.isoformat() if th_date_in else None,
                "aghd_date": aghd_date_in.isoformat() if aghd_date_in else None,
                "celebration_date": cele_date_in.isoformat() if cele_date_in else None,
                "town_hall_location": th_loc,
                "aghd_location": aghd_loc,
                "celebration_location": cele_loc,
                "currency": curr
            }
            save_settings(new_settings)
            st.success("Saved.")

def kpi_row():
    tasks = load_table("tasks", [])
    docs = load_table("documents", [])
    budget = load_table("budget", [])
    done = (tasks["status"] == "Done").sum() if "status" in tasks else 0
    total = len(tasks)
    progress = int(100 * done / total) if total else 0
    d_done = (docs["status"] == "Done").sum() if "status" in docs else 0
    d_total = len(docs)
    d_progress = int(100 * d_done / d_total) if d_total else 0
    est = float(budget["estimate"].fillna(0).sum()) if "estimate" in budget else 0.0
    act = float(budget["actual"].fillna(0).sum()) if "actual" in budget else 0.0
    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Tasks done", f"{done}/{total}", f"{progress}%")
    c2.metric("Documents complete", f"{d_done}/{d_total}", f"{d_progress}%")
    c3.metric("Estimated spend", f"{currency_symbol()}{est:,.0f}")
    c4.metric("Actual spend", f"{currency_symbol()}{act:,.0f}")

def dashboard_tab():
    st.subheader("Dashboard")
    kpi_row()
    tasks = load_table("tasks", [])
    guests = load_table("guests", [])
    colA, colB = st.columns(2)
    with colA:
        st.markdown("**Timeline (Gantt)**")
        gdf = tasks.copy()
        gdf = _ensure_date_cols(gdf, ["start","due"])
        gdf = gdf.dropna(subset=["due"])
        if not gdf.empty:
            gdf["start"] = gdf["start"].fillna(gdf["due"] - pd.to_timedelta(7, unit="d"))
            fig = px.timeline(gdf, x_start="start", x_end="due", y="task", color="milestone", hover_data=["owner","status","priority"])
            fig.update_yaxes(autorange="reversed")
            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("Add task due dates to see the timeline.")
    with colB:
        st.markdown("**Task Status**")
        if "status" in tasks and not tasks.empty:
            s = tasks["status"].value_counts().reset_index()
            s.columns = ["status","count"]
            pie = px.pie(s, names="status", values="count", hole=0.4)
            st.plotly_chart(pie, use_container_width=True)
        else:
            st.info("Add tasks to see status breakdown.")
    colC, colD = st.columns(2)
    with colC:
        st.markdown("**Guests (RSVP)**")
        if "rsvp" in guests and not guests.empty:
            g = guests["rsvp"].fillna("").replace("", "Unknown").value_counts().reset_index()
            g.columns = ["rsvp","count"]
            pie2 = px.pie(g, names="rsvp", values="count", hole=0.4)
            st.plotly_chart(pie2, use_container_width=True)
        else:
            st.info("Add guests to see RSVP chart.")
    with colD:
        st.markdown("**Priority vs Progress**")
        if not tasks.empty:
            status_map = {"Not started":0, "In progress":50, "Done":100}
            tasks["progress_%"] = tasks["status"].map(status_map).fillna(0)
            bar = px.box(tasks, x="priority", y="progress_%", points="all")
            st.plotly_chart(bar, use_container_width=True)
        else:
            st.info("Add tasks to see progress by priority.")

def kanban_tab():
    st.subheader("Kanban (by Status)")
    df = load_table("tasks", ["milestone", "task", "owner", "start", "due", "status", "priority", "notes"])
    df = _ensure_string_cols(df, ["start","due"])
    statuses = ["Not started","In progress","Done"]
    cols = st.columns(len(statuses))
    edited_frames = []
    for i, st_name in enumerate(statuses):
        with cols[i]:
            st.markdown(f"**{st_name}**")
            sub = df[df["status"] == st_name].reset_index(drop=True)
            edited = st.data_editor(sub, key=f"kanban_{st_name}", hide_index=True, num_rows="dynamic", use_container_width=True)
            edited["status"] = st_name
            edited_frames.append(edited)
    merged = pd.concat(edited_frames, ignore_index=True) if edited_frames else df
    if st.button("Save Kanban"):
        save_table("tasks", merged)
        st.success("Kanban saved.")

def tasks_tab():
    st.subheader("Tasks (Table)")
    df = load_table("tasks", ["milestone", "task", "owner", "start", "due", "status","priority","notes"])
    with st.expander("Quick add"):
        col1, col2, col3, col4 = st.columns(4)
        new_ms = col1.selectbox("Milestone", ["Town Hall","Aghd","Embassy","Celebration","Other"])
        new_task = col2.text_input("Task name")
        new_owner = col3.text_input("Owner", value="Both")
        new_due = col4.date_input("Due date", value=None)
        if st.button("Add task"):
            df.loc[len(df)] = [new_ms, new_task, new_owner, "", new_due.isoformat() if new_due else "", "Not started", "Medium", ""]
            st.success("Task added.")
    df_dt = df.copy()
    df_dt = _ensure_date_cols(df_dt, ["start","due"])
    edited = st.data_editor(
        df_dt,
        num_rows="dynamic",
        column_config={
            "start": st.column_config.DateColumn(format="YYYY-MM-DD"),
            "due": st.column_config.DateColumn(format="YYYY-MM-DD"),
            "status": st.column_config.SelectboxColumn(options=["Not started","In progress","Done"]),
            "priority": st.column_config.SelectboxColumn(options=["Low","Medium","High"]),
            "milestone": st.column_config.SelectboxColumn(options=["Town Hall","Aghd","Embassy","Celebration","Other"]),
        },
        use_container_width=True
    )
    if st.button("Save tasks (table)"):
        to_save = edited.copy()
        for c in ["start","due"]:
            if c in to_save.columns:
                to_save[c] = _iso_date_series(to_save[c])
        save_table("tasks", to_save)
        st.success("Tasks saved.")

def documents_tab():
    st.subheader("Documents & Requirements")
    st.markdown("Tick through what is needed for Town Hall, Aghd, and Embassy submission.")
    df = load_table("documents", ["category", "item", "owner", "status", "notes"])
    edited = st.data_editor(
        df,
        num_rows="dynamic",
        column_config={
            "status": st.column_config.SelectboxColumn(options=["Not started","In progress","Done"]),
            "category": st.column_config.SelectboxColumn(options=["Town Hall","Aghd","Embassy","Other"]),
        },
        use_container_width=True
    )
    if st.button("Save documents"):
        save_table("documents", edited)
        st.success("Documents saved.")

def timeline_tab():
    st.subheader("Timeline")
    st.markdown("Map what happens when (e.g. notice of marriage, Aghd appointment, venue deadlines).")
    df = load_table("timeline", ["date", "what", "where", "owner", "notes"])
    df_dt = df.copy()
    df_dt = _ensure_date_cols(df_dt, ["date"])
    edited = st.data_editor(
        df_dt,
        num_rows="dynamic",
        column_config={
            "date": st.column_config.DateColumn(format="YYYY-MM-DD"),
        },
        use_container_width=True
    )
    if st.button("Save timeline"):
        to_save = edited.copy()
        if "date" in to_save.columns:
            to_save["date"] = _iso_date_series(to_save["date"])
        save_table("timeline", to_save)
        st.success("Timeline saved.")

def budget_tab():
    st.subheader("Budget")
    st.markdown("Track estimates vs actuals and see a quick visual.")
    df = load_table("budget", ["item", "who", "estimate", "actual", "paid", "notes"])
    with st.expander("Quick add"):
        c1, c2, c3, c4 = st.columns(4)
        i_item = c1.text_input("Item (e.g. Town Hall room)")
        i_who = c2.text_input("Who (e.g. Ashkan)")
        i_est = c3.number_input("Estimate", min_value=0.0, step=50.0)
        i_act = c4.number_input("Actual", min_value=0.0, step=50.0)
        if st.button("Add budget item"):
            df.loc[len(df)] = [i_item, i_who, i_est, i_act, False, ""]
            st.success("Item added.")
    edited = st.data_editor(
        df,
        num_rows="dynamic",
        column_config={
            "estimate": st.column_config.NumberColumn(help="Estimated cost"),
            "actual": st.column_config.NumberColumn(help="Actual cost"),
            "paid": st.column_config.CheckboxColumn()
        },
        use_container_width=True
    )
    if st.button("Save budget"):
        save_table("budget", edited)
        st.success("Budget saved.")
    if not edited.empty:
        est = edited["estimate"].fillna(0).sum()
        act = edited["actual"].fillna(0).sum()
        paid = edited.loc[edited["paid"] == True, "actual"].fillna(0).sum()
        st.info(f"Total estimate: {currency_symbol()}{est:,.0f} | Actual: {currency_symbol()}{act:,.0f} | Paid: {currency_symbol()}{paid:,.0f}")
        agg = edited.melt(id_vars=["item"], value_vars=["estimate","actual"], var_name="type", value_name="amount")
        bar = px.bar(agg, x="item", y="amount", color="type", barmode="group")
        st.plotly_chart(bar, use_container_width=True)

def guests_tab():
    st.subheader("Guests")
    st.markdown("Track invites and RSVPs.")
    df = load_table("guests", ["name", "side", "invited", "rsvp", "dietary", "notes"])
    with st.expander("Quick add"):
        c1, c2, c3 = st.columns(3)
        g_name = c1.text_input("Full name")
        g_side = c2.selectbox("Side", ["Ashkan","Partner","Both","Other"])
        g_inv = c3.checkbox("Invited?", value=True)
        if st.button("Add guest"):
            df.loc[len(df)] = [g_name, g_side, g_inv, "", "", ""]
            st.success("Guest added.")
    edited = st.data_editor(
        df,
        num_rows="dynamic",
        column_config={
            "invited": st.column_config.CheckboxColumn(),
            "rsvp": st.column_config.SelectboxColumn(options=["","Yes","No","Maybe"]),
        },
        use_container_width=True
    )
    if st.button("Save guests"):
        save_table("guests", edited)
        st.success("Guests saved.")

def venues_tab():
    st.subheader("Venues & Contacts")
    st.markdown("Capture Town Hall room options, Aghd centre, reception venues, and key suppliers.")
    df = load_table("venues", ["type", "name", "contact", "phone", "email", "address", "cost_estimate", "notes"])
    edited = st.data_editor(
        df,
        num_rows="dynamic",
        column_config={
            "type": st.column_config.SelectboxColumn(options=["Town Hall","Aghd Centre","Reception","Photography","Other"]),
            "cost_estimate": st.column_config.NumberColumn(),
        },
        use_container_width=True
    )
    if st.button("Save venues"):
        save_table("venues", edited)
        st.success("Venues saved.")

def moodboard_tab():
    st.subheader("Moodboard (optional)")
    st.caption("Upload images (e.g. room, décor, outfits) and add captions to agree styles together.")
    df = load_table("moodboard", ["file","caption"])
    up = st.file_uploader("Upload images", type=["png","jpg","jpeg"], accept_multiple_files=True)
    if up:
        for file in up:
            out = DATA_DIR / f"mood_{datetime.utcnow().timestamp()}_{file.name}"
            with open(out, "wb") as f:
                f.write(file.getbuffer())
            df.loc[len(df)] = [str(out), ""]
        save_table("moodboard", df)
        st.success(f"Uploaded {len(up)} image(s).")
    if not df.empty:
        cols = st.columns(3)
        for i, row in df.iterrows():
            with cols[i % 3]:
                try:
                    st.image(row["file"], use_container_width=True)
                except Exception:
                    st.write("Image missing.")
                df.at[i,"caption"] = st.text_input("Caption", value=row.get("caption",""), key=f"cap_{i}")
        if st.button("Save captions"):
            save_table("moodboard", df)
            st.success("Captions saved.")

def exports_tab():
    st.subheader("Export")
    st.markdown("Download your tables as CSV files to share or back up.")
    for name in ["tasks","documents","timeline","budget","guests","venues","moodboard"]:
        df = load_table(name, [])
        st.download_button(
            label=f"Download {name}.csv",
            data=df.to_csv(index=False).encode("utf-8"),
            file_name=f"{name}.csv",
            mime="text/csv"
        )

def main():
    st.set_page_config(page_title=APP_TITLE, layout="wide", initial_sidebar_state="expanded")

    # 🔒 Block the app until the password is correct
    if not _check_password():
        st.stop()

    # Load settings once here so top countdown uses saved values
    settings = load_settings()
    st.session_state["settings"] = settings

    # Header (title + top metrics + sidebar details)
    header(settings)

    # Ensure CSVs exist after header (in case of first run)
    def seed_if_empty():
        docs = load_table("documents", ["category", "item", "owner", "status", "notes"])
        if docs.empty:
            save_table("documents", pd.DataFrame(DEFAULT_DOCS))
        tasks = load_table("tasks", ["milestone", "task", "owner", "start", "due", "status","priority","notes"])
        if tasks.empty:
            save_table("tasks", pd.DataFrame(DEFAULT_TASKS))
        load_table("budget", ["item", "who", "estimate", "actual", "paid", "notes"])
        load_table("guests", ["name", "side", "invited", "rsvp", "dietary", "notes"])
        load_table("venues", ["type", "name", "contact", "phone", "email", "address", "cost_estimate", "notes"])
        load_table("timeline", ["date", "what", "where", "owner", "notes"])
        load_table("moodboard", ["file","caption"])
    seed_if_empty()

    tabs = st.tabs(["Dashboard", "Kanban", "Tasks", "Documents", "Timeline",
                    "Budget", "Guests", "Venues & Contacts", "Moodboard", "Export"])

    with tabs[0]:
        set_background(BG["Dashboard"], darken=0.15)
        dashboard_tab()

    with tabs[1]:
        set_background(BG["Kanban"], darken=0.20)
        kanban_tab()

    with tabs[2]:
        set_background(BG["Tasks"], darken=0.20)
        tasks_tab()

    with tabs[3]:
        set_background(BG["Documents"], darken=0.20)
        documents_tab()

    with tabs[4]:
        set_background(BG["Timeline"], darken=0.15)
        timeline_tab()

    with tabs[5]:
        set_background(BG["Budget"], darken=0.25)
        budget_tab()

    with tabs[6]:
        set_background(BG["Guests"], darken=0.20)
        guests_tab()

    with tabs[7]:
        set_background(BG["Venues & Contacts"], darken=0.20)
        venues_tab()

    with tabs[8]:
        set_background(BG["Moodboard"], darken=0.10)
        moodboard_tab()

    with tabs[9]:
        set_background(BG["Export"], darken=0.25)
        exports_tab()

if __name__ == "__main__":
    main()
