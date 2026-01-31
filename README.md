 Correlations-for-the-Sodasense-Platform

import pandas as pd
import glob
import os

 --- ΡΥΘΜΙΣΕΙΣ ΔΙΑΔΡΟΜΗΣ ---

FOLDER_PATH = r'C:\Users\user\Desktop\CorrelationProject\1-1-2023 30-6-2023\Αγιος Γεωργιος'
TIME_COL = 'datetime'

ANALYSIS_PAIRS = [
     1. Άνεμος & Ήλιος
    ('Solar_radiation_Wm2', 'Wind_speed_ms'),
    ('Solar_radiation_Wm2', 'Wind_gust_ms'),
    ('Solar_radiation_Wm2', 'Atmospheric_temperature_C'),
    ('Wind_speed_ms', 'Wind_gust_ms'),
    
     2. Θερμοκρασία & Υγρασία
    ('Atmospheric_temperature_C', 'Atmospheric_humidity_percent'),
    ('Atmospheric_temperature_C', 'Dew_point_C'),
    ('Atmospheric_humidity_percent', 'Dew_point_C'),
    
     3. Φύλλα (Eleaf) & Περιβάλλον
    ('Eleaf_A_percent', 'Eleaf_B_percent'),
    ('Eleaf_A_percent', 'Atmospheric_humidity_percent'),
    ('Eleaf_B_percent', 'Atmospheric_humidity_percent'),
    ('Eleaf_A_percent', 'Atmospheric_temperature_C'),
    
     4. Πίεση & Βροχή
    ('Atmospheric_pressure_hPa', 'Rainfall_mm'),
    ('Atmospheric_pressure_hPa', 'Atmospheric_temperature_C'),
    
     5. CO2 & Ατμόσφαιρα (Αρχείο meteorologike1)
    ('CO2_ppm', 'Wind_speed_ms'),
    ('CO2_ppm', 'Solar_radiation_Wm2'),
    ('CO2_ppm', 'Atmospheric_temperature_C')
]

def run_master_analysis():
    try:
        os.chdir(FOLDER_PATH)
    except Exception as e:
        print(f"Σφάλμα πρόσβασης: {e}")
        return

    files = glob.glob("*.csv")
    all_results = []
    
    print(f"--- ΕΚΚΙΝΗΣΗ MASTER ΑΝΑΛΥΣΗΣ ---")
    print(f"Φάκελος: {os.path.basename(FOLDER_PATH)}")

    for f in files:
        if any(x in f for x in ["results", "Summary", "TOTAL"]): continue
        try:
            df = pd.read_csv(f, encoding='utf-8-sig', quotechar='"', skipinitialspace=True)
            df.columns = [col.replace('"', '').strip() for col in df.columns]
            
            if TIME_COL not in df.columns: continue
            
            df[TIME_COL] = pd.to_datetime(df[TIME_COL], errors='coerce')
            df = df.set_index(TIME_COL)

            for col1, col2 in ANALYSIS_PAIRS:
                if col1 in df.columns and col2 in df.columns:
                    temp = df[[col1, col2]].apply(pd.to_numeric, errors='coerce').resample('15min').mean().dropna()
                    
                    if len(temp) > 10:
                        for lag in range(-12, 13):
                            corr = temp[col1].corr(temp[col2].shift(lag))
                            all_results.append({
                                'Αρχείο': f,
                                'Συνδυασμός': f"{col1} vs {col2}",
                                'Lag_Steps': lag,
                                'Καθυστέρηση_Λεπτά': lag * 15,
                                'Correlation': round(corr, 4)
                            })
                        print(f"[OK] {f}: {col1} vs {col2}")
        except Exception as e:
            print(f"[!] Σφάλμα στο αρχείο {f}: {e}")

    if all_results:
        final_df = pd.DataFrame(all_results)
        final_df.to_csv('MASTER_RESULTS_ANALYTICAL.csv', index=False, encoding='utf-8-sig')
        
        
        final_df['Abs_Corr'] = final_df['Correlation'].abs()
        summary = final_df.sort_values('Abs_Corr', ascending=False).drop_duplicates(['Αρχείο', 'Συνδυασμός'])
        summary.drop(columns=['Abs_Corr']).to_csv('MASTER_SUMMARY_BEST_LAGS.csv', index=False, encoding='utf-8-sig')
        
        print("\n" + "="*50)
        print("Η ΑΝΑΛΥΣΗ ΟΛΟΚΛΗΡΩΘΗΚΕ!")
        print("1. MASTER_RESULTS_ANALYTICAL.csv (Όλα τα δεδομένα)")
        print("2. MASTER_SUMMARY_BEST_LAGS.csv (Τα καλύτερα αποτελέσματα)")
        print("="*50)

if __name__ == "__main__":
    run_master_analysis()
