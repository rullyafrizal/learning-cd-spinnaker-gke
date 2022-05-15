# Continuous Delivery Pipelines with Spinnaker and Kubernetes Engine

## Overview
Ini adalah catatan hasil praktik lab di Qwiklabs, dalam lab ini saya mempraktikkan bagaimana cara membuat Continous Delivery Pipeline dengan Google Kubernetes Engine, Google Cloud Source Repositories, Google Cloud Container Builder, dan Spinnaker. Di lab ini menggunakan satu contoh aplikasi sederhana yang akan kita konfigurasikan aplikasi/service ini untuk dapat di-build, test, dan deploy secara otomatis. Ketika kita modifikasi kode pada aplikasi, diharapkan perubahan ini akan otomatis trigger CD pipeline yang telah dibuat untuk rebuild, retest, dan redeploy kode ke versi terbaru secara otomatis.
<br>
<br>
**important**
<br>
Semua step yang dilakukan di catatan ini bertujuan untuk membantu kita memahami bagaimana cara membuat CD pipeline dengan Google Kubernetes Engine, Google Cloud Source Repositories, Google Cloud Container Builder, dan Spinnaker. Adapun jika ingin diterapkan di real world, maka mungkin diperlukan beberapa penyesuaian.

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
<br>
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
Pembuatan cluster biasa memakan waktu cukup lama, antara 5-10 menit<br>
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

## Konfigurasi Cloud Pub/Sub sebagai trigger Spinnaker Pipeline
1. Buat topic pub/sub dari container registry
```
gcloud pubsub topics create projects/$PROJECT/topics/gcr
```
2. Buat subscription yang bisa dibaca oleh Spinnaker, nantinya Spinnaker akan menerima notifikasi ketika image baru di-push ke registry
```
gcloud pubsub subscriptions create gcr-triggers \
    --topic projects/${PROJECT}/topics/gcr
```
3. Kita berikan Spinnaker akses ke service account untuk membaca gcr-triggers subscription
```
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:{display-name}" \
    --format='value(email)')
```
- ubah display-name sesuai dengan display nama service yang telah dibuat
```
gcloud beta pubsub subscriptions add-iam-policy-binding gcr-triggers \
    --role roles/pubsub.subscriber --member serviceAccount:$SA_EMAIL
```

## Deploy Spinnaker dengan Helm
Di bagian ini, kita akan menggunakan Helm (package manager yang digunakan untuk konfigurasi dan deploy aplikasi kubernetes) untuk deploy Spinnaker.

### Konfigurasi Helm
1. Berikan Helm role `cluster-admin`
```
kubectl create clusterrolebinding user-admin-binding \
    --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```
2. Berikan Spinnaker role `cluster-admin` juga, agar bisa deploy resource ke semua namespaces
```
kubectl create clusterrolebinding --clusterrole=cluster-admin \
    --serviceaccount=default:default spinnaker-admin
```
3. Tambahkan stable chart deploymnt ke repositories helm
```
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

## Konfigurasi Spinnaker
1. Masih di dalam Cloud Shell, kita akan buat bucket untuk menyimpan data konfigurasi pipeline Spinnaker
```
export PROJECT=$(gcloud info \
    --format='value(config.project)')
```
```
export BUCKET=$PROJECT-spinnaker-config
```
```
gsutil mb -c regional -l us-central1 gs://$BUCKET
```
2. Jalankan command ini untuk membuat `spinnaker-config.yaml`, yang nantinya digunakan untuk mendeskripsikan bagaimana Helm nantinya harus menginstall Spinnaker
```
export SA_JSON=$(cat spinnaker-sa.json)
export PROJECT=$(gcloud info --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
cat > spinnaker-config.yaml <<EOF
gcs:
  enabled: true
  bucket: $BUCKET
  project: $PROJECT
  jsonKey: '$SA_JSON'
dockerRegistries:
- name: gcr
  address: https://gcr.io
  username: _json_key
  password: '$SA_JSON'
  email: 1234@5678.com
# Disable minio as the default storage backend
minio:
  enabled: false
# Configure Spinnaker to enable GCP services
halyard:
  spinnakerVersion: 1.19.4
  image:
    repository: us-docker.pkg.dev/spinnaker-community/docker/halyard
    tag: 1.32.0
    pullSecrets: []
  additionalScripts:
    create: true
    data:
      enable_gcs_artifacts.sh: |-
        \$HAL_COMMAND config artifact gcs account add gcs-$PROJECT --json-path /opt/gcs/key.json
        \$HAL_COMMAND config artifact gcs enable
      enable_pubsub_triggers.sh: |-
        \$HAL_COMMAND config pubsub google enable
        \$HAL_COMMAND config pubsub google subscription add gcr-triggers \
          --subscription-name gcr-triggers \
          --json-path /opt/gcs/key.json \
          --project $PROJECT \
          --message-format GCR
EOF
```

### Deploy Spinnaker Chart
1. Gunakan Helm command-line interface untuk deploy chart dengan set konfigurasi kita
```
helm install -n default cd stable/spinnaker -f spinnaker-config.yaml \
           --version 2.0.0-rc9 --timeout 10m0s --wait
```
- Proses di atas cukup lama, bisa memakan waktu antara 5-8 menit
2. Setelah selesai, jalankan command di bawah ini untuk set up port forwarding ke Spinnaker melalui Cloud Shell
```
export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" \
    -o jsonpath="{.items[0].metadata.name}")
```
```
kubectl port-forward --namespace default $DECK_POD 8080:9000 >> /dev/null &
```
3. Untuk membuka Spinnaker UI, klik icon Web Preview pada cloud shell dan pilih `Preview on port 8080`, selanjutnya akan terbuka browser baru dan akan terbuka Spinnaker UI

## Build Docker Image
- Di bagian ini, kita akn mengkonfigurasi Cloud Build untuk mendeteksi perubahan pada kode, dan secara otomatis akan build Docker image, dan push ke Container Registry

## Buat Source Code Repository (Git)
1. Download contoh kodingan aplikasi
```
gsutil -m cp -r gs://spls/gsp114/sample-app.tar .
```
2. Unpack source code
```
mkdir sample-app
tar xvf sample-app.tar -C ./sample-app
```
3. Change Directories
```
cd sample-app
```
4. Set username dan email address pada git commit, ubah [USERNAME] dengan username yang telah dibuat sebelumnya
```
git config --global user.email "$(gcloud config get-value core/account)"
```
```
git config --global user.name "[USERNAME]"
```
5. Jalankan initial commit
- `git init`
- `git add .`
- `git commit -m "Initial commit"`
6. Buat repository
```
gcloud source repos create sample-app
```
```
git config credential.helper gcloud.sh
```
7. Tambahkan repository yang telah dibuat sebagai remote
```
export PROJECT=$(gcloud info --format='value(config.project)')
```
```
git remote add origin https://source.developers.google.com/p/$PROJECT/r/sample-app
```
8. Jalankan command untuk push ke remote
```
git push origin master
```
9. Cek source code dengan navigate ke **Source Repositories**
10. Pilih **sample-app**

### Konfigurasi build trigger
![Build Trigger](/assets/build-trigger.png)
1. Navigate ke **Navigation Menu** > **Cloud Build** > **Triggers**.
2. Pilih **Create Trigger**
3. Pasang konfigurasi sesuai di bawah ini
```
Name: sample-app-tags

Event: Push new tag

Select your newly created sample-app repository.

Tag: v1.*

Configuration: Cloud Build configuration file (yaml or json)

Cloud Build configuration file location: /cloudbuild.yaml
```
4. Klik Create
<br>

Mulai sekarang, kapanpun kita push sebuah tag baru dengan prefic "v", secara otomatis Container Builder akan men-build dan push aplikasi sebagai Docker image ke Container Registry

### Menyiapkan Kubernetes Manifests untuk digunakan di Spinnaker
Spinnaker butuh akses ke K8s manifests agar bisa deploy ke cluster. Di bagian ini, kita akan membuat Cloud Storage Bucket yang akan diisi dengan manifests selama proses CI di dalam Cloud Build. Setelah manifest tersimpan di Cloud Storage, Spinnaker bisa men-download dan apply selama eksekusi pipeline
1. Create Bucket
```
export PROJECT=$(gcloud info --format='value(config.project)')
```
```
gsutil mb -l us-central1 gs://$PROJECT-kubernetes-manifests
```
2. Enable versioning di bucket agar nanti kita bisa punya history tracker dari manifest
```
gsutil versioning set on gs://$PROJECT-kubernetes-manifests
```
3. Set project ID di dalam K8s deployment manifest
```
sed -i s/PROJECT/$PROJECT/g k8s/deployments/*
```
4. Commit perubahan
```
git commit -a -m "Set project ID"
```

### Build Image
1. Di dalam cloud shell, di directory `sample-app` jalankan command ini
```
git tag v1.0.0
```
2. Push tag
```
git push --tags
```
3. Cloud Build otomatis akan build perubahan kode kita ke yang terbaru, ini bisa dilihat di bagian History di halaman Cloud Build

## Konfigurasi Deployment Pipelines
- Setelah secara otomatis build Docker image, kita perlu deploy image yang telah di-build ke dalam Kubernetes Cluster

### Install spin CLI untuk managing Spinnaker
1. Download `spin` CLI
```
curl -LO https://storage.googleapis.com/spinnaker-artifacts/spin/1.14.0/linux/amd64/spin
```
2. Make `spin` executable
```
chmod +x spin
```

### Buat deployment pipeline
1. Gunakan `spin` untuk membuat aplikasi di Spinnaker. Jangan lupa untuk set email pemilik juga
```
./spin application save --application-name sample \
                        --owner-email "$(gcloud config get-value core/account)" \
                        --cloud-providers kubernetes \
                        --gate-endpoint http://localhost:8080/gate
```
Di tutorial ini, pipeline diatur sedemikian rupa agar bisa mendeteksi ketika sebuah Docker image dengan prefix tag "v" telah sampai di Container Registry.
2. Dari `sample-app` directory, jalankan command ini untuk upload pipeline ke Spinnaker instance
```
export PROJECT=$(gcloud info --format='value(config.project)')
sed s/PROJECT/$PROJECT/g spinnaker/pipeline-deploy.json > pipeline.json
./spin pipeline save --gate-endpoint http://localhost:8080/gate -f pipeline.json
```

## Trigger dan Pantau Eksekusi Pipeline Secara Manual
Konfigurasi yang telah kita buat menggunakan notifikasi push tag. Di bagian sebelumnya, kita telah push tag ke Cloud Soure Repositories yang itu telah men-trigger Cloud Build untuk men-build dan push image ke Container Registry. Nah untuk memverifikasi pipeline, kita bisa trigger secara manual.

1. Navigate ke Spinnaker UI, caranya bisa dengan pilih Web Preview di Cloud Shell dan pilih `Preview on Port 8080`
2. Setelah masuk ke Spinnaker UI, pilih **Application** > **sample**
![Spinnaker UI 1](/assets/spinnaker-ui-1.png)
3. Klik **Pipelines** yang ada di atas
4. Klik **Start Manual Execution**, setelah itu **Run**
![Spinnaker UI 2](/assets/spinnaker-ui-2.png)
5. Klik **Execution Details** untuk melihat detail informasi dari progress pipeline
![Spinnaker UI 3](/assets/spinnaker-ui-3.png)
Progress bar menunjukkan status deployment dari pipeline dan step deployment. <br>
![Progress Bar](/assets/spinnaker-ui-4.png)
Warna biru berarti sedang berjalan, hijau berarti telah selesai, dan merah artinya gagal.
6. Klik salah satu dtep di progress bar untuk melihat detail. Setelah 3 sampai 5 menit, fase integration tes telah selesaai dan butuh manual approval untuk lanjut deploy
7. Hover di bagian kuning dan klik **Continue**
![Spinnaker UI 5](/assets/spinnaker-ui-5.png)
8. Untuk melihat aplikasinya, pilih **Infrastructure** > **Load Balancers**
9. Scroll terus ke bawah dan klik Default, di bawah `service sample-frontend-application` kita akan melihat details dari load balancer yang muncul di sisi kana halaman. Jika tidak ada, cukup refresh halaman sampai muncul
10. Scroll sisi navigasi di kanan hingga ke bawah, dan buka IP Address dari Ingress di tab baru browser.
11. Ketika halaman sudah muncul maka akan muncul versi canary dari aplikasi, terkadang juga akan muncul versio production juga
![Spinnaker UI 6](/assets/spinnaker-ui-6.png)

## Trigger pipeline dari perubahan kode
Di bagian ini kita akan test pipeline secara e2e dengan membuat perubahan di kode, push Git tag, dan memantau pipeline berjalan. Dengan push tag prefix "v", kita akan men-trigger Container Builder untuk build Docker image baru dan push ke Container Registry. Spinnaker akan secara otomatis mendeteksi image baru yang memiliki tag prefix "v" dan men-trigger pipeline untuk deploy image tadi ke container, menjalankan test, dan roll out image ke semua pod.

1. Di dalam `sample-app` directory, ubah warna dari frontend aplikasi kita tadi dari orange ke biru
```
sed -i 's/orange/blue/g' cmd/gke-info/common-service.go
```
2. Tag dan commit perubahan kita
```
git commit -a -m "Change color to blue"
``` 
```
git tag v1.0.1
```
```
git push --tags
```
3. Di dalam Console, **Cloud Build** > **History**, tunggu beberapa menit hingga build terbaru muncul
4. Kembali ke Spinnaker UI dan klik **Pipelines** untuk memantau pipeline mulai berjalan deploy image. Triggered pipeline akan butuh waktu beberapa menit untuk muncul

## Observe Canary Deployment
Ketika deployment dihentikan sementara (paused), kembali ke web page yang menampilkan aplikasi kita dan coba untuk refresh beberapa kali. Hal ini akan memunculkan versi canary dari aplikasi.
<br>

Ketika pipeline telah selesai, aplikasi kita akan terlihat seperti ini dengan warna biru sesuai perubahan kita yang terakhir tadi
![Spinnaker UI 7](/assets/spinnaker-ui-7.png)
Di tahap ini, berarti kita telah sukses untuk roll out aplikasi kita ke environment prod

3. **Optional**, kita bisa rollback perubahan apabila terjadi sesuatu yang tidak diinginkan.
```
git revert v1.0.1
```
Tekan **CTRL+O**,**ENTER**,**CTRL+X**
```
git tag v1.0.2
```
```
git push --tags
```
4. Ketika build dan pipeline selesai, kita tinggal verifikasi rollback dengan cara navigate ke Spinnaker UI melalui **Infrastructure** > **Load Balancers**, kemudian klik **service sample-frontend-productionn**, lalu scroll ke bawah dan klik **Default**, akses ingress IP-nya
5. Jika berhasil, aplikasi kita akan kembali berwarna orange secara UI.