# TUGAS UTS PEMROGRAMAN WEB

# NAMA : MUHAMMAD RAKHA GHANI
# NIM : 312410421
# KELAS : I241C

## 📑 𝗧𝗔𝗕𝗟𝗘 𝗢𝗙 𝗖𝗢𝗡𝗧𝗘𝗡𝗧𝗦
1. [Executive Summary](#1-executive-summary)
2. [System Architecture & Database Blueprint](#2-system-architecture--database-blueprint)
3. [Vulnerability Analysis (Celah Keamanan)](#3-vulnerability-analysis-celah-keamanan)
4. [Exploitation Methodology (Metode Serangan)](#4-exploitation-methodology-metode-serangan)
5. [Defensive Countermeasure (Mitigasi)](#5-defensive-countermeasure-mitigasi)
6. [Visual Evidence (Log Sistem)](#6-visual-evidence-log-sistem)
7. [References](#7-references)

---

## 1. EXECUTIVE SUMMARY
Repositori ini berisi *source code* dan dokumentasi dari pengujian penetrasi (Pen-Test) skala kecil pada sistem autentikasi web. Fokus utama riset ini adalah mendemonstrasikan **CWE-89: Improper Neutralization of Special Elements used in an SQL Command** (SQL Injection). Eksperimen membuktikan bahwa sistem login yang dibangun dengan `mysqli_query` dinamis tanpa sanitasi dapat dibobol sepenuhnya menggunakan manipulasi operator logika `OR`.

---

## 2. SYSTEM ARCHITECTURE & DATABASE BLUEPRINT
Pengujian dilakukan pada *environment* lokal (XAMPP). Berikut adalah skema *database* `eksperimen_keamanan` yang digunakan sebagai target operasi:

```sql
-- DDL & DML untuk target database
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(50) NOT NULL
);

-- Menyisipkan data rahasia target (Admin)
INSERT INTO users (username, password) VALUES ('admin', 'rahasia123');
```

---

## 3. VULNERABILITY ANALYSIS (CELAH KEAMANAN)
File `vulnerable.php` mensimulasikan sistem yang rentan. Kerentanan terjadi karena input dari pengguna (`$_POST`) langsung digabungkan (*concatenated*) ke dalam string SQL tanpa proses validasi, *escaping*, atau sanitasi.

**Code Breakdown (Bagian Rentan):**
```php
$username = $_POST['username'];
$password = $_POST['password'];

// ❌ CRITICAL FLAW: Input langsung dimasukkan ke dalam query
$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
$result = $conn->query($query);
```

---

## 4. EXPLOITATION METHODOLOGY (METODE SERANGAN)
Untuk mengeksploitasi celah di atas, serangan dilakukan pada kolom input `username` untuk memanipulasi *parser* MySQL.

* **Attack Vector / Payload:** `' OR '1'='1'#`
* **Dummy Password:** `hacker123`

**Proses di Database Engine:**
Ketika payload dikirimkan, *query* yang ditangkap oleh server berubah menjadi seperti ini:
```sql
SELECT * FROM users WHERE username = '' OR '1'='1'# AND password = 'hacker123'
```
**Analisis Logika:**
1.  `'` di awal menutup string untuk parameter username.
2.  `OR '1'='1'` adalah kondisi *tautology* (selalu benar/TRUE).
3.  `#` adalah simbol komentar dalam MySQL. Ini membutakan mesin database, sehingga perintah pengecekan `AND password = 'hacker123'` diabaikan sepenuhnya.
4.  **Hasil:** Server mengembalikan *TRUE* dan memberikan akses masuk sebagai pengguna pertama di tabel (Admin).

---

## 5. DEFENSIVE COUNTERMEASURE (MITIGASI)
Untuk menambal (*patching*) kerentanan *Zero-day* ini, file `secure.php` diimplementasikan menggunakan arsitektur **Prepared Statements**.

**Code Breakdown (Bagian Aman):**
```php
// ✅ SECURE APPROACH: Menggunakan Prepared Statements
$stmt = $conn->prepare("SELECT * FROM users WHERE username = ? AND password = ?");

// Mengikat parameter ('ss' berarti 2 data string). Input pengguna dianggap murni sebagai DATA, bukan PERINTAH SQL.
$stmt->bind_param("ss", $username, $password); 
$stmt->execute();
$result = $stmt->get_result();
```
Dengan arsitektur ini, *database engine* memisahkan struktur *query* dari data. Meskipun pengguna memasukkan simbol berbahaya seperti `'` atau `#`, itu hanya akan dibaca sebagai teks biasa, menetralisir ancaman SQLi secara absolut.

---

## 6. VISUAL EVIDENCE (LOG SISTEM)
> *Klik pada setiap dropdown untuk memperluas tangkapan layar dari hasil eksperimen.*

<details>
<summary><b>🟢 LOG 01: NORMAL AUTHENTICATION (SYSTEM STABLE)</b></summary>
<br>
Sistem memproses kredensial yang valid dengan benar. Logika <i>query</i> berjalan sesuai desain.<br><br>

> *(Masukkan link gambar normal login di sini)*

<img width="598" height="341" alt="login normal" src="https://github.com/user-attachments/assets/a45c2416-8d14-4d4c-9d52-828e573d93c5" />

</details>

<details>
<summary><b>🔴 LOG 02: SYSTEM BREACHED (SQL INJECTION SUCCESS)</b></summary>
<br>
<b>CRITICAL ALERT:</b> Injeksi logika OR sukses dieksekusi. Autentikasi berhasil di-bypass total tanpa mengetahui <i>password</i> yang sebenarnya.<br><br>

> *(Masukkan link gambar pembobolan di sini)*

<img width="394" height="208" alt="login bobol" src="https://github.com/user-attachments/assets/b820a80a-af1c-44b4-b698-e2a4d03f4eb9" />

</details>

<details>
<summary><b>🛡️ LOG 03: SYSTEM SECURED (ATTACK BLOCKED)</b></summary>
<br>
Serangan SQLi ditangkis. Payload berbahaya direduksi menjadi string biasa oleh <i>Prepared Statements</i>. Akses ditolak.<br><br>

> *(Masukkan link gambar aman/gagal login di sini)*

<img width="500" height="273" alt="login gagal" src="https://github.com/user-attachments/assets/b3a45191-13fa-42d2-9446-36151b16897a" />


</details>

---
