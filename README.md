# Excavator-Hishob
import streamlit as st
import pandas as pd
import os
import datetime
import io

st.set_page_config(page_title="एक्सीवेटर हिशोब व मेंटेनन्स बुक", layout="wide")

st.title("🚜 एक्सीवेटर (पोकलेन) मशीन हिशोब व मेंटेनन्स बुक")

# इंटरनेटवरून एक्सीवेटरचा फोटो दाखवणे
st.image("https://unsplash.com", 
         caption="तुमचा डिजिटल एक्सीवेटर मॅनेजर", use_container_width=True)

DB_FILE = "excavator_data.xlsx"
CONFIG_FILE = "service_config.xlsx"

REQUIRED_COLUMNS = [
    "तारीख", "मशीन नंबर", "ऑपरेटरचे नाव", "ग्राहकाचे नाव", "भाड्याचा प्रकार", 
    "सुरुवातीचे HMR", "अंतिम HMR", "एकूण तास", 
    "दर (प्रति तास)", "एकूण bill", "ॲडव्हान्स रक्कम", "बाकी रक्कम", 
    "डिझेल (लिटर)", "डिझेल दर (प्रति लिटर)", "डिझेल एकूण खर्च", "इतर खर्च"
]

# १. मुख्य डेटाबेस लोड करणे
if os.path.exists(DB_FILE):
    try:
        df = pd.read_excel(DB_FILE)
        df["तारीख"] = pd.to_datetime(df["तारीख"]).dt.date
        for col in REQUIRED_COLUMNS:
            if col not in df.columns:
                df[col] = "" if col in ["मशीन नंबर", "ऑपरेटरचे नाव"] else 0
    except Exception as e:
        df = pd.DataFrame(columns=REQUIRED_COLUMNS)
else:
    df = pd.DataFrame(columns=REQUIRED_COLUMNS)

# २. ऑईल चेंज कॉन्फिगरेशन फाईल लोड करणे
cfg_df = pd.DataFrame(columns=["मशीन_नंबर", "शेवटची_सर्व्हिस_तारीख", "शेवटचे_सर्व्हिस_HMR"])
if os.path.exists(CONFIG_FILE):
    try:
        cfg_df = pd.read_excel(CONFIG_FILE)
        cfg_df["मशीन_नंबर"] = cfg_df["मशीन_नंबर"].astype(str).str.strip()
    except:
        pass

# 💎 स्मार्ट मल्टिपल मशीन यादी मेकर (Bug Fix for Dropdown)
all_machines_set = set()

# कामाच्या डेटाबेसमधून मशीन नंबर गोळा करणे
if not df.empty and "मशीन नंबर" in df.columns:
    for m in df["मशीन नंबर"].dropna().unique():
        if str(m).strip() != "":
            all_machines_set.add(str(m).strip())

# ऑईल चेंज ट्रॅकर फाईलमधून देखील सर्व मशीन नंबर गोळा करणे
if not cfg_df.empty and "मशीन_नंबर" in cfg_df.columns:
    for m in cfg_df["मशीन_नंबर"].dropna().unique():
        if str(m).strip() != "":
            all_machines_set.add(str(m).strip())

# सेटचे रूपांतर यादीत (List) करणे
available_machines = sorted(list(all_machines_set))

# जर अजून एकही नोंद नसेल तर डिफॉल्ट मशीन देणे
if not available_machines:
    available_machines = ["मशीन १", "मशीन २"]

# साईडबार - रोजची नवीन नोंद करा
st.sidebar.header("📝 रोजची नवीन नोंद करा")
with st.sidebar.form("entry_form", clear_on_submit=True):
    date = st.date_input("तारीख", datetime.date.today())
    machine_no = st.text_input("मशीन नंबर (उदा. MH-12-AB-1234)").strip()
    operator_name = st.text_input("ऑपरेटरचे नाव")
    customer = st.text_input("ग्राहकाचे / साईटचे नाव")
    rent_type = st.selectbox("भाड्याचा प्रकार", ["डिझेलसहित", "विदाऊट डिझेल"])
    start_hmr = st.number_input("सुरुवातीचे HMR Reading", min_value=0.0, step=0.1)
    end_hmr = st.number_input("अंतिम HMR Reading", min_value=0.0, step=0.1)
    rate = st.number_input("ठरलेला दर (प्रति तास)", min_value=0, step=100)
    advance = st.number_input("ॲडव्हान्स मिळालेली रक्कम", min_value=0, step=500)
    
    st.sidebar.markdown("---")
    st.sidebar.subheader("⛽ डिझेल व इतर खर्च")
    diesel_liters = st.number_input("डिझेल भरले (लिटर)", min_value=0.0, step=1.0)
    diesel_rate = st.number_input("डिझेल दर (प्रति लिटर रू.)", min_value=0.0, step=1.0)
    other_cost = st.number_input("इतर खर्च / ऑपरेटर भत्ता (रू.)", min_value=0, step=50)
    
    submit = st.form_submit_button("डेटा सेव्ह करा")

if submit:
    if end_hmr < start_hmr: 
        st.sidebar.error("❌ अंतिम HMR कमी असू शकत नाही!")
    elif customer == "": 
        st.sidebar.error("❌ कृपया ग्राहकाचे नाव टाका!")
    elif machine_no == "":
        st.sidebar.error("❌ कृपया मशीन नंबर टाका!")
    else:
        total_hours = round(end_hmr - start_hmr, 2)
        total_bill = int(total_hours * rate)
        balance = int(total_bill - advance)
        calculated_diesel_cost = int(diesel_liters * diesel_rate)
        
        new_row = {
            "तारीख": date, "मशीन नंबर": machine_no, "ऑपरेटरचे नाव": operator_name,
            "ग्राहकाचे नाव": customer, "भाड्याचा प्रकार": rent_type, 
            "सुरुवातीचे HMR": start_hmr, "अंतिम HMR": end_hmr, "एकूण तास": total_hours, 
            "दर (प्रति तास)": rate, "एकूण bill": total_bill, "ॲडव्हान्स रक्कम": advance, "बाकी रक्कम": balance, 
            "डिझेल (लिटर)": diesel_liters, "डिझेल दर (प्रति लिटर)": diesel_rate, 
            "डिझेल एकूण खर्च": calculated_diesel_cost, "इतर खर्च": other_cost
        }
        
        df = pd.concat([df, pd.DataFrame([new_row])], ignore_index=True)
        df.to_excel(DB_FILE, index=False)
        st.sidebar.success("✅ नोंद यशस्वीरित्या जतन केली!")
        st.rerun()

# 🚨 मुख्य स्क्रीनवर अलर्ट पाहण्यासाठी मशीन निवडणे (आता मल्टिपल पर्याय दिसतील)
st.markdown("---")
selected_alert_machine = st.selectbox("🚨 कोणत्या मशीनचा सर्व्हिसिंग अलर्ट पाहायचा आहे?", available_machines)

# निवडलेल्या मशीनचा शेवटचा ऑईल चेंज डेटा शोधणे
last_service_date = datetime.date.today()
last_service_hmr = 0.0

if not cfg_df.empty:
    match = cfg_df[cfg_df["मशीन_नंबर"] == str(selected_alert_machine).strip()]
    if not match.empty:
        try:
            last_service_date = pd.to_datetime(match["शेवटची_सर्व्हिस_तारीख"].values[0]).date()
            last_service_hmr = float(match["शेवटचे_सर्व्हिस_HMR"].values[0])
        except:
            pass

# 🔧 साईडबारमध्ये ऑईल चेंज फॉर्म
st.sidebar.markdown("---")
st.sidebar.subheader("🛠️ नवीन ऑईल चेंजची नोंद")
with st.sidebar.form("service_form"):
    s_machine = st.text_input("ऑईल बदललेली मशीन नं.", value=str(selected_alert_machine)).strip()
    s_date = st.date_input("ऑईल बदललेली तारीख", last_service_date)
    s_hmr = st.number_input("ऑईल बदलले वेळचे HMR", min_value=0.0, value=last_service_hmr, step=0.1)
    service_submit = st.form_submit_button("ऑईल चेंज डेटा अपडेट करा")

if service_submit:
    if s_machine == "":
        st.sidebar.error("❌ कृपया मशीन नंबर टाका!")
    else:
        # जुनी नोंद काढून नवीन अपडेट करणे
        if not cfg_df.empty:
            cfg_df = cfg_df[cfg_df["मशीन_नंबर"] != str(s_machine)]
        new_cfg = pd.DataFrame([{"मशीन_नंबर": s_machine, "शेवटची_सर्व्हिस_तारीख": s_date, "शेवटचे_सर्व्हिस_HMR": s_hmr}])
        cfg_df = pd.concat([cfg_df, new_cfg], ignore_index=True)
        cfg_df.to_excel(CONFIG_FILE, index=False)
        st.sidebar.success(f"✅ मशीन {s_machine} चा डेटा सेव्ह झाला!")
        st.rerun()


# 🛠️ सर्व्हिसिंग आणि ऑईल चेंज रिमाइंडर विभाग
st.subheader(f"🚨 मशीन सर्व्हिसिंग अलर्ट: {selected_alert_machine}")

latest_hmr = 0.0
if not df.empty:
    df["मशीन नंबर"] = df["मशीन noble" if "मशीन noble" in df.columns else "मशीन नंबर"].astype(str).str.strip()
    machine_df = df[df["मशीन नंबर"] == str(selected_alert_machine).strip()]
    if not machine_df.empty:
        latest_hmr = pd.to_numeric(machine_df["अंतिम HMR"], errors='coerce').max()
        if pd.isna(latest_hmr): latest_hmr = 0.0

col_box1, col_box2 = st.columns(2)
with col_box1:
    st.info(f"⚙️ मशीनचे सध्याचे चालू रीडिंग: *{latest_hmr} HMR* आहे.")
with col_box2:
    st.info(f"📅 शेवटचे ऑईल चेंज तारीख: *{last_service_date.strftime('%d-%m-%Y')}* ला *{last_service_hmr} HMR* वर झाले होते.")

next_service_hmr = last_service_hmr + 700.0
hours_left = round(next_service_hmr - latest_hmr, 1)

col_rem1, col_rem2 = st.columns(2)
with col_rem1:
    st.warning(f"🔧 पुढील ऑईल चेंज *{next_service_hmr} HMR* ला करणे गरजेचे आहे.")
with col_rem2:
    if hours_left <= 0:
        st.error(f"🚨 अलार्म: सर्व्हिसिंगची वेळ *टळून गेली आहे!* मशीन {abs(hours_left)} तास जास्त चालली आहे. त्वरित ऑईल बदला!")
    elif hours_left <= 20:
        st.error(f"⚠️ अलार्म: सर्व्हिसिंगची वेळ झाली आहे! फक्त *{hours_left} तास* शिल्लक आहेत. ऑईल बदलून घ्या!")
    else:
        st.success(f"✅ अजून *{hours_left} तास* मशीन बिन्धास्त चालेल. त्यानंतर ऑईल चेंज करा.")


# मुख्य स्क्रीनवर सर्च आणि तारीख फिल्टर
if not df.empty:
    st.markdown("---")
    st.subheader("🔍 प्रगत हिशोब फिल्टर (Advanced Search & Date Filter)")
    
    col1, col2, col3 = st.columns(3)
    with col1:
        search_machine = st.text_input("🚜 मशीन नंबरनुसार फिल्टर", value="")
    with col2:
        search_operator = st.text_input("👨‍✈️ ऑपरेटरच्या नावानुसार फिल्टर")
    with col3:
        min_date = min(df["तारीख"]) if not df.empty else datetime.date.today()
        max_date = max(df["तारीख"]) if not df.empty else datetime.date.today()
        date_range = st.date_input("📅 तारीख कालावधी निवडा", [min_date, max_date])
    
    filtered_df = df.copy()
    if search_machine:
        filtered_df = filtered_df[filtered_df["मशीन नंबर"].astype(str).str.contains(search_machine, case=False, na=False)]
    if search_operator:
        filtered_df = filtered_df[filtered_df["ऑपरेटरचे नाव"].astype(str).str.contains(search_operator, case=False, na=False)]
    
    if isinstance(date_range, list) or isinstance(date_range, tuple):
        if len(date_range) == 2:
            filtered_df = filtered_df[(filtered_df["तारीख"] >= date_range[0]) & (filtered_df["तारीख"] <= date_range[1])]

    t_bill = pd.to_numeric(filtered_df["एकूण bill"], errors='coerce').fillna(0).sum()
    t_adv = pd.to_numeric(filtered_df["ॲडव्हान्स रक्कम"], errors='coerce').fillna(0).sum()
    t_bal = pd.to_numeric(filtered_df["बाकी रक्कम"], errors='coerce').fillna(0).sum()
    t_exp = pd.to_numeric(filtered_df["डिझेल एकूण खर्च"], errors='coerce').fillna(0).sum() + pd.to_numeric(filtered_df["इतर खर्च"], errors='coerce').fillna(0).sum()
    
    m1, m2, m3, m4 = st.columns(4)
    m1.metric("💰 निवडलेला एकूण बिझनेस", f"रू. {int(t_bill):,}")
    m2.metric("📥 एकूण जमा (ॲडव्हान्स)", f"रू. {int(t_adv):,}")
    m3.metric("🔴 एकूण बाकी (Balance)", f"रू. {int(t_bal):,}")
    m4.metric("📈 निव्वळ नफा (Net Profit)", f"रू. {int(t_bill - t_exp):,}")
    
    st.subheader("📋 फिल्टर केलेला हिशोब तक्ता")
    st.dataframe(filtered_df, use_container_width=True)
