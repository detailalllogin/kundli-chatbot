from fastapi import FastAPI, Request
import swisseph as swe
from fastapi.middleware.cors import CORSMiddleware
from fpdf import FPDF
from fastapi.responses import FileResponse
import os
import matplotlib.pyplot as plt
import numpy as np
import httpx

app = FastAPI()

# Allow frontend to connect (CORS policy)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

PROKERALA_API_KEY = "your_api_key_here"

@app.post("/kundli")
async def get_kundli(request: Request):
    data = await request.json()

    year, month, day = map(int, data['dob'].split('-'))
    hour, minute = map(int, data['tob'].split(':'))
    lat = float(data['lat'])
    lon = float(data['lon'])

    utc_hour = hour + minute / 60.0 - 5.5
    jd = swe.julday(year, month, day, utc_hour)

    planets = [swe.SUN, swe.MOON, swe.MARS, swe.MERCURY, swe.JUPITER, swe.VENUS, swe.SATURN, swe.RAHU, swe.KETU]
    planet_names = ['Sun', 'Moon', 'Mars', 'Mercury', 'Jupiter', 'Venus', 'Saturn', 'Rahu', 'Ketu']

    positions = {}
    longitudes = []
    for i, planet in enumerate(planets):
        pos = swe.calc_ut(jd, planet)[0]
        positions[planet_names[i]] = round(pos, 2)
        longitudes.append(pos)

    flag = swe.FLG_SWIEPH | swe.FLG_SPEED
    house_data, ascmc = swe.houses(jd, lat, lon, 'P')
    ascendant = round(ascmc[0], 2)

    moon_long = positions['Moon']
    rashi_index = int(moon_long // 30)
    rashis = ['मेष', 'वृषभ', 'मिथुन', 'कर्क', 'सिंह', 'कन्या', 'तुला', 'वृश्चिक', 'धनु', 'मकर', 'कुंभ', 'मीन']
    moon_rashi = rashis[rashi_index]

    # Chart creation
    fig, ax = plt.subplots(figsize=(6, 6), subplot_kw={'projection': 'polar'})
    ax.set_theta_direction(-1)
    ax.set_theta_zero_location("E")
    ax.set_yticklabels([])
    ax.set_xticks(np.radians(np.linspace(0, 330, 12)))
    ax.set_xticklabels(rashis)
    for i, angle in enumerate(longitudes):
        angle_rad = np.radians(angle)
        ax.plot([angle_rad], [1], marker='o', label=planet_names[i])
        ax.text(angle_rad, 1.1, planet_names[i], fontsize=8, ha='center')

    chart_filename = f"chart_{year}{month}{day}_{hour}{minute}.png"
    chart_path = f"./{chart_filename}"
    plt.legend(bbox_to_anchor=(1.05, 1), loc='upper left')
    plt.savefig(chart_path, bbox_inches='tight')
    plt.close()

    # PDF generation with embedded chart
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=14)
    pdf.cell(200, 10, txt="आपकी कुंडली", ln=True, align='C')
    pdf.ln(10)
    pdf.cell(200, 10, txt=f"लग्न (Ascendant): {ascendant}°", ln=True)
    pdf.cell(200, 10, txt=f"चंद्र राशि: {moon_rashi}", ln=True)
    pdf.ln(5)
    for planet, pos in positions.items():
        pdf.cell(200, 10, txt=f"{planet}: {pos}°", ln=True)
    pdf.ln(10)
    pdf.image(chart_path, x=30, y=None, w=150)

    filename = f"kundli_{year}{month}{day}_{hour}{minute}.pdf"
    filepath = f"./{filename}"
    pdf.output(filepath)

    # Basic planetary yoga/interpretation (demo)
    yogas = []
    if positions['Sun'] > 270 and positions['Sun'] < 300:
        yogas.append("सूर्य मकर राशि में है — आत्म-विश्वास और स्थिरता")
    if positions['Moon'] > 60 and positions['Moon'] < 90:
        yogas.append("चंद्र मिथुन में — मानसिक सक्रियता और संप्रेषण क्षमता")
    if abs(positions['Sun'] - positions['Moon']) < 12:
        yogas.append("सूर्य-चंद्र नजदीक — अमावस्या/सूर्य ग्रहण योग")

    return {
        "positions": positions,
        "julian_day": jd,
        "ascendant": ascendant,
        "moon_sign": moon_rashi,
        "pdf_file": filename,
        "chart_file": chart_filename,
        "yogas": yogas
    }

@app.post("/prokerala/kundli")
async def get_kundli_prokerala(request: Request):
    data = await request.json()

    params = {
        "ayanamsa": 1,
        "coordinates": {
            "latitude": data["lat"],
            "longitude": data["lon"]
        },
        "datetime": f'{data["dob"]}T{data["tob"]}:00+05:30',
    }

    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://api.prokerala.com/v2/astrology/birth-details",
            headers={"Authorization": f"Bearer {PROKERALA_API_KEY}"},
            json=params
        )

    return response.json()

@app.get("/kundli/pdf/{filename}")
async def download_pdf(filename: str):
    filepath = f"./{filename}"
    if os.path.exists(filepath):
        return FileResponse(filepath, media_type='application/pdf', filename=filename)
    return {"error": "File not found"}

@app.get("/kundli/chart/{filename}")
async def download_chart(filename: str):
    filepath = f"./{filename}"
    if os.path.exists(filepath):
        return FileResponse(filepath, media_type='image/png', filename=filename)
    return {"error": "Chart not found"}
