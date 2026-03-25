# AWS PrivateLink Solution: Overlapping CIDR Challenge

![AWS Architecture Diagram](Overlapping CIDR VPC Peering.drawio.png)

## 📌 Overview
Project ini mendokumentasikan implementasi **AWS PrivateLink** sebagai solusi efektif dan *scalable* untuk mengatasi konflik rute pada VPC dengan CIDR identik (**VPC A & VPC C: 10.0.0.0/16**). Pendekatan ini dipilih karena kebutuhan spesifik berupa *service access* (HTTP ke Nginx), bukan konektivitas jaringan umum (*full mesh network*) antar VPC.

## 💡 Arsitektur Logic (The Golden Answer)
> **Mengapa Tidak Terjadi Konflik Routing?**
> AWS PrivateLink bekerja pada **Layer 4 (TCP)** dan tidak mengandalkan tabel rute VPC tradisional (Layer 3) antar VPC. Koneksi dilakukan melalui **Interface Endpoint (ENI)** yang memiliki IP lokal di masing-masing VPC Consumer. Karena trafik dialirkan melalui **AWS Backbone Network** ke Network Load Balancer (NLB) di VPC Provider, overlapping CIDR antara VPC A dan VPC C tidak relevan bagi sistem routing.

---

## I. Persiapan Infrastruktur Network

### A. Provider Side (VPC B - 10.1.0.0/16)
1. **Deployment Nginx Server:**
   - Instance EC2 diletakkan di Private Subnet untuk keamanan maksimal.
   - **Security Group:** Inbound **TCP 80** hanya diizinkan dari **CIDR Subnet tempat NLB berada**, memastikan akses hanya melalui jalur resmi Load Balancer.
2. **Target Group:**
   - Protocol: **TCP** | Port: **80**.
   - Target: Nginx EC2 Instance.

### B. Consumer Side (VPC A & VPC C - 10.0.0.0/16)
1. **Deployment Client Instance:**
   - EC2 Instance di masing-masing VPC (Client A & Client C) sebagai *service consumer*.

---

## II. Implementasi PrivateLink & Load Balancing

### Step 1: Membuat Internal NLB (VPC B)
- **Scheme:** Internal (Terisolasi dari Internet).
- **Subnet:** Dikonfigurasi pada Private Subnet menggunakan ENI per Availability Zone.
- **Listener:** TCP 80 meneruskan trafik ke Target Group Nginx.

### Step 2: Membuat Endpoint Service (VPC B)
- Menghubungkan **NLB-PrivateLink** sebagai penyedia layanan.
- **Acceptance required:** Diaktifkan untuk verifikasi keamanan setiap koneksi masuk.
- Menyalin **Service Name** unik untuk digunakan oleh Consumer.

### Step 3: Membuat Interface Endpoint (VPC A & VPC C)
- **Service Category:** Menggunakan *Service Name* dari VPC B.
- **Security Group Interface Endpoint:**
  - **Inbound:** Allow TCP 80 dari Security Group EC2 Client atau CIDR lokal.
  - **Outbound:** Allow All (Default).
- **Private DNS:** Fitur *Private DNS Name* diaktifkan agar layanan dapat diakses melalui domain kustom (contoh: `nginx.service.local`).

---

## III. Pengujian & Verifikasi (L4 Stateful)

### A. Verifikasi Koneksi
1. Admin di VPC B menyetujui (*Accept*) permintaan koneksi dari VPC A dan C.
2. Gunakan `curl` dari sisi Client untuk memastikan *Handshake* L4 dan *Response* L7 berhasil:
   ```bash
   # Verifikasi detail koneksi dan HTTP response
   curl -Iv http://<vpce-dns-name-atau-private-dns>

### B. Hasil Analisis
- Client A dan Client C mendapatkan respon sukses secara independen.
- Source IP: Log Nginx di VPC B akan mencatat IP dari Node NLB sebagai sumber trafik. Response tetap bersifat dua arah (stateful TCP), namun inisiasi koneksi hanya dapat dilakukan oleh Consumer.

## IV. Trade-offs Analysis
### Kelebihan (Pros)
- Kelebihan (Pros)Kekurangan (Cons)Overlapping Resolution: Solusi paling bersih untuk CIDR bentrok.
- High Security: Tidak ada eksposur network secara penuh antar VPC.
- Managed Service: Mengurangi kompleksitas operasional routing manual.

### Kekurangan (Cons)
- Cost: Biaya per jam untuk NLB dan Interface Endpoint per AZ.
- Initiation: Hanya mendukung inisiasi koneksi dari Consumer ke Provider.
- Service Only: Tidak dirancang untuk full mesh network communication antar VPC.

## V. Alternative Approach
Alternatif solusi lainnya adalah menggunakan NAT Gateway atau NAT Instance untuk melakukan translasi IP sebelum paket masuk ke VPC Provider. Namun, pendekatan tersebut jauh lebih kompleks untuk dikelola dan kurang aman dibandingkan abstraksi service-level yang ditawarkan oleh AWS PrivateLink.\

## 🏁 Kesimpulan
Implementasi ini menunjukkan bagaimana Service-level Abstraction memungkinkan komunikasi antar VPC tanpa bergantung pada routing berbasis CIDR. Solusi ini berhasil mengatasi keterbatasan overlapping network dengan pendekatan yang aman, scalable, dan sesuai dengan standar industri profesional.

