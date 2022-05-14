# Continuous Delivery Pipelines with Spinnaker and Kubernetes Engine

## Overview
Ini adalah catatan hasil praktik lab di Qwiklabs, dalam lab ini saya mempraktikkan bagaimana cara membuat Continous Delivery Pipeline dengan Google Kubernetes Engine, Google Cloud Source Repositories, Google Cloud Container Builder, dan Spinnaker. Di lab ini menggunakan satu contoh aplikasi sederhana yang akan kita konfigurasikan aplikasi/service ini untuk dapat di-build, test, dan deploy secara otomatis. Ketika kita modifikasi kode pada aplikasi, diharapkan perubahan ini akan otomatis trigger CD pipeline yang telah dibuat untuk rebuild, retest, dan redeploy kode ke versi terbaru secara otomatis.

## Objektif
- Memasang environment dengan Google Cloud Shell, membuat cluster K8s dari Google Cloud Shell, dan konfigurasi skema identity and user management.
- Download aplikasi contoh sederhana, dan menaruhnya di Git Repository dengan menggunakan Google Cloud Source Repository
- Deploy Spinnaker ke Kubernetes Engine dengan menggunakan package manager Helm
- Build Docker image
- Buat trigger untuk build docker image ketika terjadi perubahan kode pada Git Repository
- Mengkonfigurasi Spinnaker pipeline agar bisa deploy aplikasi ke Kubernetes Engine secara berkelanjutan (continuously)
- Deploy perubahan kode, trigger pipeline, dan rilis ke production

## Diagram Pipeline Architecture
![Pipeline Architecture](/assets/pipeline-arch.png)
Untuk bisa deliver aplikasi secara berkelanjutan, kita butuh sebuah proses automasi yang dapat build, test, dan update software. Perubahan dalam kode aplikasi seharusnya bisa ter-otomatisasi atau istilahnya mengalir di dalam pipeline yang sudah termasuk: 
- Pembuatan artifak/image, 
- Unit testing, 
- Functional testing, dan 
- Deployment ke production.
Jadi ketika terjadi perubahan kode, pipeline yang kita bangun akan otomatis melakukan 4 hal di atas.<br>

Dalam suatu kasus jika kita ingin perubahan kode hanya berlaku ke beberapa user, maka kita bisa menerapkan **Canary Deployment**. Sehingga dengan menggunakan prinsip deployment ini, kita bisa roll-back ke versi sebelumnya jika terjadi sesuatu ataupun fitur terbaru pada aplikasi kita tidak memenuhi kepuasan para pengguna.
<br>

Dengan Kubernetes Engine dan Spinnaker, kita bisa membuat flow Continuous Delivery yang dapat membantu untuk selalu memastikan bahwa aplikasi yang kita kirim ke production akan tetap berjalan dengan baik. Memastikan sebuah aplikasi yang kita kirim ke production akan tetap berjalan dengan baik adalah hal yang sangat penting. Akan sangat melelahkan apabila kita harus melakukan hal yang sama berulang-ulang secara manual, maka dengan otomatisasi ini, kita cukup memantau perubahan dari aplikasi kita bisa pass dari otomatisasi ini. Setelah dirasa bahwa aplikasi kita dapat berjalan dengan baik, dan siap untuk di-deploy ke production, kita bisa men-trigger pipeline untuk men-deploy ke production.

## Diagram Application Delivery Pipeline
Dalam lab ini kita akan membuat Continuous Delivery Pipeline sesuai dengan flow yang ada di diagram ini
![Pipeline Flow](/assets/pipeline-flow.png)

## Setup Environment
1. Set default compute zone terlebih dahulu
```
gcloud config set compute/zone {zone}
```
2. Buat cluster K8s dengan GKE
```
gcloud container clusters create {nama-cluster} \
    --machine-type={type-machine}
```
Pembuatan cluster biasa memakan waktu cukup lama, antara 5-10 menit
3. Ketika selesai, output yang diharapkan adalah sebuah report/laporan yang berisi nama, lokasi, versi, IP Address, machine-type, node version, jumlah node, dan status cluster yang mengindikasikan bahwa cluster sedang berjalan

## Konfigurasi Identity and Access Management
Kita konfigurasi dengan membuat sebuah **service account** di Cloud Identity Access Management (Cloud IAM) untuk mendelegasikan akses ke Spinnaker, sehingga nantinya spinnaker bisa menyimpan data ke Cloud Storage. Mengapa harus disimpan di Cloud Storage? Hal ini untuk memastikan reliability dan resiliency dari pipeline itu sendiri. Jika suatu saat deployment-nya gagal, kita bisa buat deployment kembali dalam waktu singkat dengan akses ke pipeline yang sama dengan aslinya.

1. Buat service account
```
gcloud iam service-accounts create {nama-service-account} \
    --display-name {display-name-sa}
```
2. Simpan email address service account dan project ID saat ini ke env variable untuk nantinya digunakan di command lain
```
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:{display-name-sa}" \
    --format='value(email)')
```
```
export PROJECT=$(gcloud info --format='value(config.project)')
```
3. Pasang `storage.admin` role ke service account tadi
```
gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin \
    --member serviceAccount:$SA_EMAIL
```
4. Download service-account key, yang nantinya akan dipakai untuk install Spinnaker
```
gcloud iam service-accounts keys create spinnaker-sa.json \
     --iam-account $SA_EMAIL
```
- Contoh Output. <br>
`created key [12f224e036437704b91a571792462ca6fc4cd438] of type [json] as [spinnaker-sa.json] for [spinnaker-account@qwiklabs-gcp-gcpd-f5e16da10e5d.iam.gserviceaccount.com]`




