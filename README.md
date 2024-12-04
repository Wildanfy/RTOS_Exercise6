# RTOS-EXERCISE 6
## Deskripsi
Program ini adalah implementasi sistem multitasking menggunakan FreeRTOS pada platform STM32. Dua tugas paralel, yaitu **GreenLEDTask** dan **RedLEDTask**, berfungsi untuk mengakses sumber daya bersama yang dilindungi menggunakan mekanisme **taskENTER_CRITICAL** dan **taskEXIT_CRITICAL** untuk menghindari kontensi sumber daya. Selain itu, program ini menggunakan GPIO untuk mengontrol tiga LED sebagai indikasi aktivitas tugas dan kontensi.
### **Tujuan**
1. Mengimplementasikan sistem multitasking pada STM32 menggunakan FreeRTOS.
2. Menghindari kontensi sumber daya bersama menggunakan mekanisme **Critical Section**.
3. Memberikan indikasi visual terhadap aktivitas dan potensi kontensi.

---

## Pin IOC
![Exercise 6](https://github.com/user-attachments/assets/45d20600-90d9-4e65-8ad7-d3694e4c801b)
### **Struktur Program**

1. **Tugas Hijau (GreenLEDTask)**:
   - Menyalakan LED hijau, mengakses sumber daya bersama, lalu mematikannya.
   - Berjalan dengan prioritas lebih rendah (`osPriorityIdle`).

2. **Tugas Merah (RedLEDTask)**:
   - Menyalakan LED merah, mengakses sumber daya bersama, lalu mematikannya.
   - Berjalan dengan prioritas normal (`osPriorityNormal`).

3. **Akses Sumber Daya Bersama**:
   - Diimplementasikan melalui fungsi `accessSharedData()`, yang memanipulasi variabel global `startFlag`.
   - Variabel `startFlag` digunakan untuk mendeteksi kontensi.
   - Simulasi operasi baca/tulis dilakukan dengan delay menggunakan `__asm("nop")`.

4. **Indikasi Kontensi**:
   - Jika dua tugas mencoba mengakses sumber daya bersama secara bersamaan (tanpa perlindungan), LED biru menyala.

---


#### **1. Konfigurasi GPIO**

Fungsi `MX_GPIO_Init()` menginisialisasi tiga pin GPIO untuk LED:
- **GPIO_PIN_0**: LED hijau untuk tugas hijau.
- **GPIO_PIN_1**: LED merah untuk tugas merah.
- **GPIO_PIN_2**: LED biru untuk indikasi kontensi.

```c
HAL_GPIO_WritePin(GPIOA, LED1_Pin|LED2_Pin|LED3_Pin, GPIO_PIN_RESET);
```

Semua LED awalnya dimatikan.

---

#### **2. Inisialisasi FreeRTOS**

FreeRTOS diatur untuk menjalankan tiga tugas:
- **defaultTask**: Tugas default yang tidak memiliki fungsi spesifik.
- **GreenLEDTask**: Menangani kontrol LED hijau.
- **RedLEDTask**: Menangani kontrol LED merah.

```c
osThreadDef(GreenLEDTask, green_led, osPriorityIdle, 0, 128);
osThreadDef(RedLEDTask, red_led, osPriorityNormal, 0, 128);
```

---

#### **3. Implementasi Tugas**

- **GreenLEDTask**:
  - Menyalakan LED hijau.
  - Memasuki critical section untuk mengakses sumber daya bersama.
  - Mematikan LED hijau setelah selesai.
  - Delay selama 500 ms sebelum iterasi berikutnya.

```c
void green_led(void const * argument) {
    for(;;) {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_SET); // Nyalakan LED hijau
        taskENTER_CRITICAL();                              // Masuk critical section
        accessSharedData();
        taskEXIT_CRITICAL();                               // Keluar critical section
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_RESET); // Matikan LED hijau
        osDelay(500);                                      // Delay 500 ms
    }
}
```

- **RedLEDTask**:
  - Mirip dengan GreenLEDTask tetapi menggunakan LED merah dan delay 100 ms.

```c
void red_led(void const * argument) {
    for(;;) {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET); // Nyalakan LED merah
        taskENTER_CRITICAL();                              // Masuk critical section
        accessSharedData();
        taskEXIT_CRITICAL();                               // Keluar critical section
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET); // Matikan LED merah
        osDelay(100);                                      // Delay 100 ms
    }
}
```

---

#### **4. Fungsi `accessSharedData()`**

Fungsi ini mensimulasikan akses ke sumber daya bersama menggunakan `startFlag` untuk mendeteksi potensi kontensi:
1. Jika `startFlag == 1`, sumber daya tersedia, dan `startFlag` diubah menjadi 0.
2. Jika `startFlag == 0`, LED biru menyala untuk menunjukkan kontensi.
3. Delay dilakukan untuk mensimulasikan waktu akses sumber daya.
4. `startFlag` diatur kembali ke 1, dan LED biru dimatikan.

```c
void accessSharedData(void) {
    if (startFlag == 1) {
        startFlag = 0; // Tandai sumber daya sedang diakses
    } else {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_SET); // Nyalakan LED biru untuk indikasi kontensi
    }
    SimulateReadWriteOperation();
    startFlag = 1; // Sumber daya tersedia kembali
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_RESET); // Matikan LED biru
}
```

---

#### **5. Simulasi Operasi Baca/Tulis**

Fungsi `SimulateReadWriteOperation()` menggunakan loop delay untuk mensimulasikan proses akses sumber daya.

```c
void SimulateReadWriteOperation(void) {
    volatile uint32_t delay_count = 0;
    const uint32_t delay_target = 1000000;
    for (delay_count = 0; delay_count < delay_target; delay_count++) {
        __asm("nop");
    }
}
```

---

### **Output Program**

1. **LED Hijau dan Merah**:
   - Menyala bergantian untuk menunjukkan aktivitas tugas hijau dan merah.
   - Frekuensi nyala:
     - Hijau: Setiap 500 ms.
     - Merah: Setiap 100 ms.

2. **LED Biru**:
   - Menyala hanya jika terjadi kontensi sumber daya.

---

### **Kesimpulan**

1. **Kontensi Sumber Daya**:
   - Mekanisme `taskENTER_CRITICAL` dan `taskEXIT_CRITICAL` memastikan akses eksklusif ke sumber daya bersama, mencegah kontensi.

2. **Indikasi Visual**:
   - LED digunakan untuk memantau aktivitas tugas dan mendeteksi kesalahan dalam pengelolaan sumber daya.

3. **FreeRTOS Multitasking**:
   - Program ini memberikan gambaran nyata tentang bagaimana FreeRTOS menangani multitasking dengan prioritas tugas yang berbeda.

## Video Demo
https://github.com/user-attachments/assets/c1cc71fa-743f-425b-9f6e-e7ec41057120
### Contributor
- Wildan Faizin (3222600011)
- Dhanang Fadhila Trisnandi (3222600015)
