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





