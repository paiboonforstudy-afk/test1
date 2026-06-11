**1. Clone the repository**
```bash
git clone <repo-url>
cd ride-hailing-project
```

**2. Create virtual environment**
```bash
python -m venv .venv
.venv\Scripts\activate       # Windows
source .venv/bin/activate    # Mac / Linux
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Set up environment variables**
```bash
cp .env.example .env
# Fill in your Azure credentials
```

**5. Start Nominatim Docker**
```bash
docker run -p 8080:8080 mediagis/nominatim
```

**6. Generate data**
```bash
python data_generator.py historical --count 5000 --format csv --duration 2026-01-01:2026-02-01
```

**7. Upload to Azure**
```bash
python upload_historical.py
```