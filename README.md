# ⚡ ESP32 Solar Grid-Tie Zero Export Limiter & Inverter Power Throttle

[![Lisensi: MIT](https://img.shields.io/badge/Lisensi-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform: ESP32](https://img.shields.io/badge/Platform-ESP32%20|%20FreeRTOS-blue.svg)](#)
[![Framework: Arduino IDE](https://img.shields.io/badge/Framework-Arduino%20IDE%202.0%2B-teal.svg)](https://www.arduino.cc/)
[![Status: Firmware Produksi](https://img.shields.io/badge/Status-Firmware%20Produksi-brightgreen.svg)](#)
[![Developer: Muhammad Fikri](https://img.shields.io/badge/Developer-Muhammad%20Fikri-blue.svg)](#)

Zero-export solar power manager monitoring grid import/export via high-speed CT clamp, throttling grid-tie inverters via Modbus RS485 / SunSpec in under 200ms.

---

## 📊 Diagram Blok Arsitektur & Skema Alur Rangkaian

Visualisasi interaktif alur daya, akuisisi sinyal sensor, pemrosesan algoritma inti, dan aktuasi proteksi perangkat:

```mermaid
graph TD
    subgraph Power_Source ["⚡ Sumber Energi & Jaringan"]
        PV["Panel Surya / Baterai / AC Mains"] --> SENS_V["Sensor Tegangan (ZMPT101B / Divider)"]
        PV --> SENS_I["Sensor Arus (ACS712 / Shunt)"]
        SENS_V -->|"Analog Signal"| MCU["🧠 ESP32 (Dual-Core Xtensa LX6)"]
        SENS_I -->|"Analog Signal"| MCU
    end

    subgraph Energy_Processing ["🧠 Algoritma & Power DSP"]
        MCU -->|"Discrete Sampling"| TRMS["True-RMS & Active Power Math"]
        TRMS -->|"MPPT / State Engine"| CTRL["P&O MPPT / Load Shedding Logic"]
        CTRL -->|"Fast PWM (Timer1/MCPWM)"| GATE["Gate Driver Stage"]
    end

    subgraph Load_Protection ["🔋 Manajemen Beban & Storage"]
        GATE -->|"Switching MOSFET/SSR"| LOAD["Beban Listrik / Bank Baterai"]
        MCU -->|"I2C Visual"| DISP["Layar Monitor Daya Real-Time"]
        MCU -->|"Cloud Telemetry"| CLOUD["MQTT / Modbus TCP Stream"]
    end

    style MCU fill:#1565c0,stroke:#0d47a1,stroke-width:2px,color:#fff
    style TRMS fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#fff
    style GATE fill:#bf360c,stroke:#870000,stroke-width:2px,color:#fff
```

---

## 📦 Daftar Komponen & Bahan Lengkap (Bill of Materials - BOM)

Berikut rincian spesifikasi komponen fisik dan modul yang dibutuhkan untuk membangun proyek ini:

| No | Nama Komponen / Modul | Estimasi Jumlah | Fungsi & Spesifikasi Teknis |
|:---|:---|:---|:---|
| 1 | **ESP32 Dev Module (30/38-Pin)** | 1 Unit | Mikrokontroler Dual-Core dengan FreeRTOS, Wi-Fi 2.4GHz & BLE |
| 2 | **Regulator Step-Down LM2596 / PSU 5V 2A** | 1 Unit | Sumber daya listrik 5V DC teregulasi |
| 3 | **Sensor Arus Hall Effect ACS712 / INA219 / Shunt Resistor** | 1-2 Unit | Pengukuran arus DC/AC presisi tinggi |
| 4 | **Modul Sensor Tegangan ZMPT101B / Precision Voltage Divider** | 1-2 Unit | Pengukuran tegangan True-RMS jaringan listrik |
| 5 | **Power MOSFET N-Channel (IRFZ44N / FDP047N10) / SSR Relay** | 2-4 Unit | Sakelar daya switching kontrol beban dan MPPT |
| 6 | **Induktor Daya & Kapasitor Filter Low-ESR** | 1 Set | Komponen penyaring riak daya switching konverter |
| 7 | **Layar OLED SSD1306 / LCD 20x4 I2C** | 1 Unit | Tampilan metrik Watt, Volt, Ampere, dan Efisiensi |

---

## 🧠 Arsitektur Sistem & Fitur Utama

- **FreeRTOS Multi-Core Priority Scheduling:** Memisahkan pemrosesan sinyal presisi tinggi dari task telemetri untuk mencegah *latency jitter*.
- **Digital Signal Processing (DSP) & Filtering:** Dilengkapi algoritma digital filtering terdedikasi untuk eliminasi derau sinyal analog.
- **Non-Volatile Storage (NVS Preferences Flash):** Parameter kalibrasi, *setpoint*, dan konfigurasi tersimpan secara persisten terhadap siklus pemadaman daya.
- **Hardware Failsafe & Emergency Interlock:** Perlindungan otomatis jika terjadi anomali tegangan, kelebihan beban arus, atau pemicuan tombol *Emergency Stop*.
- **Industrial Telemetry & Diagnostics:** Pelaporan status operasional secara real-time via Serial/JSON stream.

---

## 🔌 Skema Pinout & Koneksi Hardware

| Komponen / Sinyal | Pin (ESP32) | Deskripsi Fungsi |
|:---|:---|:---|
| **Sensor Analog Input** | `GPIO 36 (ADC1)` | Jalur pembacaan sensor utama berpresisi tinggi |
| **Emergency Stop (E-Stop)** | `GPIO 34` | Pemicu pengaman darurat hardware interrupt |
| **Actuator / Relay Utama** | `GPIO 26` | Pengendali beban daya tinggi / relay aktuator |
| **Acoustic Alarm Buzzer** | `GPIO 25` | Indikator peringatan audible saat terjadi anomali |
| **Status / Heartbeat LED** | `GPIO 2` | Indikator status aktivitas sistem real-time |

---

## 🛠️ Panduan Perakitan Hardware (Langkah Demi Langkah)

1. **Persiapan Catu Daya:** Hubungkan catu daya utama ke jalur daya mikrokontroler. Pasang kapasitor *decoupling* 100nF di dekat pin VCC untuk meredam ripple switching.
2. **Pemasangan Sensor & Modul:** Sambungkan jalur sinyal sensor ke pin mikrokontroler yang telah ditentukan. Gunakan resistor pull-up 4.7kΩ pada jalur SDA/SCL jika menggunakan modul I2C.
3. **Pemasangan Aktuator:** Hubungkan modul relay / gate driver MOSFET ke pin kontrol output. Pasang dioda *flyback* (1N4007) pada beban induktif untuk mengeliminasi lonjakan tegangan balik (*back-EMF*).
4. **Pemasangan Tombol Emergency Stop:** Sambungkan tombol darurat ke pin interupsi eksternal dengan konfigurasi *Active-LOW* menggunakan resistor *pull-up*.
5. **Verifikasi Koneksi:** Lakukan pengecekan jalur ground bersama (*Common Ground*) pada seluruh modul sebelum menyalakan daya.

---

## 🚀 Panduan Kompilasi & Upload (Arduino IDE)

1. Buka **Arduino IDE 2.0+**.
2. Masuk ke menu **Tools > Board**:
   * Pilih **`ESP32 Dev Module`**.
3. Pastikan dependensi pustaka terpasang via Library Manager:
   * `ArduinoJson`
   * `Wire` & `SPI`
   * `Preferences`
4. Buka berkas [`esp32-gridtie-limiter-zero-export.ino`](./esp32-gridtie-limiter-zero-export.ino).
5. Klik tombol **Verify** (✓) kemudian **Upload** (➔).
6. Buka **Serial Monitor** pada baudrate **`115200`** untuk melihat streaming telemetri dan status operasional.

---

## 📄 Lisensi
Didistribusikan di bawah lisensi open-source **MIT License**. Dikembangkan oleh **Muhammad Fikri**.
