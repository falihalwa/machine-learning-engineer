# Demo Guide: Agent Platform
Panduan praktis langkah demi langkah dari penyiapan data, pelatihan AutoML, hingga pengujian prediksi langsung (*online prediction*).

### Langkah 1: Siapkan Dataset dari BigQuery Publik

Gunakan dataset Census Income untuk klasifikasi biner (`<=50K` vs `>50K`).

1. Buka **Google Cloud Console** > masuk ke menu **Agent Platform** (atau **Vertex AI**).
2. Di panel navigasi sebelah kiri, pilih **Datasets** lalu klik **+ Create**.
3. Isi parameter dataset:
    - **Dataset name:** `census_automl_demo`
    - **Select a data type and objective:** Pilih kartu **Tabular** > **Regression / Classification**.
    - **Region:** Pilih region standar, misalnya `us-central1`.
    - Klik **Create**.
4. Di bagian *Select a data source*, pilih **Select a table or view from BigQuery**.
5. Masukkan path tabel publik berikut ke kolom input:Plaintext
    
    ```
    bigquery-public-data.ml_datasets.census_adult_income
    ```
    
6. Klik **Continue**. Tunggu 1–2 menit sampai proses inspeksi data selesai dan tabel ringkasan kolom muncul.

### Langkah 2: Konfigurasi dan Jalankan AutoML Training

1. Di halaman detail dataset, klik tombol **Train new model** di bar bagian atas.
2. **Training method:**
    - Pastikan memilih opsi **AutoML** (bukan *Custom training*).
    - Klik **Continue**.
3. **Model details:**
    - **Model name:** `model_census_automl`
    - **Target column:** Pilih kolom `income_bracket` (kolom ini berisi label kelas target).
    - Klik **Continue**.
4. **Features & Transformations:**
    - Biarkan seluruh kolom fitur terpilih secara default.
    - Klik **Continue**.
5. **Compute and pricing:**
    - **Budget:** Masukkan nilai minimum `1` node hour (cukup untuk eksperimen awal).
    - Centang opsi **Enable early stopping** agar job berhenti otomatis saat metrik validasi konvergen.
6. Klik **Start training**.
    - Status awal akan berlabel `Pending`, lalu berubah menjadi `Running`. Tunggu hingga selesai (`Succeeded`).

### Langkah 3: Deploy Model ke Online Endpoint

Setelah status job di menu **Training** selesai, model perlu di-*deploy* ke endpoint agar bisa menerima request JSON secara real-time.

1. Buka panel **Model Registry** di sidebar.
2. Klik nama model yang baru selesai dilatih (`model_census_automl`).
3. Buka tab **Deploy & Test** di bagian atas, lalu klik **Deploy to endpoint**.
4. Konfigurasi endpoint:
    - **Endpoint name:** Biarkan default (`model_census_automl_endpoint`) atau buat baru.
    - **Traffic split:** 100%.
    - **Machine type:** Pilih mesin standar terendah (misalnya `n1-standard-2`).
    - **Number of compute nodes:** Atur minimum `1` dan maksimum `1`.
5. Klik **Deploy**. Proses pembuatan VM endpoint membutuhkan waktu sekitar 10–15 menit hingga statusnya menjadi **Active**.

### Langkah 4: Uji Coba Model dan Masukkan Input Data

Setelah status endpoint aktif, Anda bisa menguji prediksi langsung lewat konsol atau REST API.

**Cara A: Test Langsung via GUI Konsol (Paling Mudah)**

1. Tetap di halaman model pada tab **Deploy & Test**.
2. Gulir ke bawah ke bagian **Test your model**.
3. Konsol akan menyediakan form input berbasis field atau panel JSON raw.
4. Masukkan contoh payload data uji berikut ke dalam form:
    - `age`: `38`
    - `workclass`: `Private`
    - `education`: `Bachelors`
    - `education_num`: `13`
    - `marital_status`: `Never-married`
    - `occupation`: `Exec-managerial`
    - `relationship`: `Not-in-family`
    - `race`: `White`
    - `sex`: `Male`
    - `capital_gain`: `0`
    - `capital_loss`: `0`
    - `hours_per_week`: `45`
    - `native_country`: `United-States`
5. Klik tombol **Predict**.
6. Di panel kanan, hasil prediksi akan langsung muncul menampilkan probabilitas tiap kelas (misalnya: `{"<=50K": 0.23, ">50K": 0.77}`).

**Cara B: Test via Cloud Shell (cURL / REST API)**

Buka **Cloud Shell** (ikon terminal di bar atas konsol) dan jalankan:

1. Buat file input `request.json`:Bash
    
    ```
    cat <<EOF > request.json
    {
       "instances": [
          {
             "age": "38",
             "workclass": "Private",
             "education": "Bachelors",
             "education_num": "13",
             "marital_status": "Never-married",
             "occupation": "Exec-managerial",
             "relationship": "Not-in-family",
             "race": "White",
             "sex": "Male",
             "capital_gain": "0",
             "capital_loss": "0",
             "hours_per_week": "45",
             "native_country": "United-States"
          }
       ]
    }
    EOF
    ```
    
2. Eksekusi request inferensi:Bash
    
    ```
    ENDPOINT_ID=$(gcloud ai endpoints list --region=us-central1 --filter="displayName:model_census_automl_endpoint" --format="value(name)")
    
    curl -X POST \
      -H "Authorization: Bearer $(gcloud auth print-access-token)" \
      -H "Content-Type: application/json" \
      https://us-central1-aiplatform.googleapis.com/v1/${ENDPOINT_ID}:predict \
      -d @request.json
    ```
    

### Langkah 5: Bersihkan Resource (Cleanup)

Endpoint yang berjalan terus akan memakan biaya sewa VM. Setelah selesai mencoba:

1. Buka menu **Online prediction** (atau **Endpoints**).
2. Klik nama endpoint Anda.
3. Di tabel model yang di-deploy, klik ikon titik tiga (opsi) di sisi kanan model > pilih **Undeploy model from endpoint**.
4. Setelah proses undeploy selesai, hapus endpoint tersebut dengan memilih **Delete endpoint**.
