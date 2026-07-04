# Silabus Modul — Kubernetes Fundamental (arah: DevOps/SRE)

Panduan detail per modul untuk tutor. Status & recap per user ada di `LEARNING.md` — file ini
adalah *kurikulum*, bukan log. Satu "sesi" dirancang ±1 jam, tapi pace mengikuti user
(sesi boleh dipecah/digabung). Urutan modul disusun supaya tiap modul memakai hasil modul
sebelumnya, dan semuanya memakai manifest yang sudah ada di repo ini.

Prinsip lintas modul (dari [Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)):
manifest deklaratif di git, `apply` bukan perintah imperatif, label selalu dipakai, satu objek
per file boleh digabung kalau satu kesatuan aplikasi, jangan pakai `latest` di production.

---

## Modul 0 — Baseline & Orientasi ✅ (selesai)

Observability dasar: `get` / `describe` / `logs` / `get events`; memetakan node, versi,
namespace sistem vs namespace kerja. Recap ada di `LEARNING.md`.

## Modul 1 — Workload Fundamental ✅ (selesai)

Deployment + Service (NodePort), Service sebagai L4 load balancer, IP Pod yang fana,
scheduling & taint, rolling update/rollback, diagnosa ImagePullBackOff. Recap di `LEARNING.md`.

---

## Modul 2 — Konfigurasi: ConfigMap & Secret

**Kenapa penting untuk DevOps/SRE:** memisahkan config dari image adalah dasar 12-factor;
salah kelola Secret adalah salah satu insiden keamanan paling umum.

**File repo:** `configmap/configmap-1..3.yaml`, `secret/secret-1..4.yaml`,
`deployment/deployment-configmap-1..2.yaml`, `deployment/deployment-secret-1..2.yaml`,
`pod/pod-with-cm.yaml`, `pod/pod-with-secret.yaml`.

**Dokumentasi:**
[ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) ·
[Secret](https://kubernetes.io/docs/concepts/configuration/secret/) ·
[Secret best practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

### Sesi 2a — ConfigMap sebagai env & sebagai file
1. Apply `configmap-1` (key-value) → pakai lewat `deployment-configmap-1` (`env` +
   `configMapKeyRef`; varian `envFrom` ada di komentar — coba keduanya, bahas kapan pilih mana).
2. Apply `configmap-2` (`index.html` utuh) → mount sebagai file lewat `pod/pod-with-cm.yaml`
   (Pod `demo-app`), buktikan nginx menyajikan halaman custom (`kubectl port-forward` + `curl`).
3. `configmap-3` (config.yaml + features.json): pola "config file aplikasi" yang paling umum
   di production.
4. **Eksperimen kunci:** ubah nilai ConfigMap lalu amati — env var **tidak** berubah di Pod
   berjalan; file yang di-mount berubah *eventually* (kecuali `subPath`). Kesimpulan production:
   perubahan config butuh `kubectl rollout restart` (atau hash config di pod template).

### Sesi 2b — Secret
1. Bandingkan `secret-1` (`data`, base64) vs `secret-2` (`stringData`, plaintext) — objek
   akhirnya identik. **Tunjukkan `base64 -d`: base64 itu encoding, bukan enkripsi.**
2. `secret-3`/`secret-4` (service-account JSON): pola mount credential file, umum untuk
   akses cloud provider.
3. Konsumsi via `deployment-secret-2` dan `pod/pod-with-secret.yaml`.
4. **Drill sabotase bawaan repo:** `deployment-secret-1` mereferensikan Secret `app-secret`
   yang tidak ada → Pod `CreateContainerConfigError`. User harus menemukan sendiri akar
   masalahnya lewat `describe`, lalu memperbaiki nama referensi.

### Sesi 2c — Praktik production
- RBAC membatasi siapa yang bisa `get secret`; etcd encryption at rest; kenapa perusahaan
  pakai external secret manager (Vault/External Secrets Operator) — cukup konsep, tanpa install.
- `immutable: true` untuk ConfigMap/Secret besar & stabil (performa + proteksi salah edit).
- Jangan commit Secret asli ke git — hubungkan dengan `.gitignore` repo ini.

**Checkpoint:** user bisa menjelaskan env vs mount + implikasi update-nya; mendemokan
base64 ≠ enkripsi; menyelesaikan drill `CreateContainerConfigError` < 10 menit.

---

## Modul 3 — Storage: PVC, PV & StorageClass

**File repo:** `pvc/pvc.yaml` (PVC `app-storage`), `pod/pod-with-pvc.yaml`.

**Dokumentasi:**
[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) ·
[Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) ·
[local-path-provisioner](https://github.com/rancher/local-path-provisioner)

### Sesi 3a — Kenapa PVC Pending (belajar dari kegagalan yang disengaja)
1. Cluster ini **tidak punya StorageClass** — apply `pvc.yaml` dan amati `Pending`.
   `describe` → "no storage class is set". Ini bukan bug; ini pelajaran rantai
   PVC → StorageClass → provisioner → PV.
2. Bedakan static provisioning (admin bikin PV manual) vs dynamic (StorageClass + provisioner).

### Sesi 3b — Pasang provisioner & pakai storage
1. Install `local-path-provisioner` (proyek Rancher, cocok dengan RKE2) — latihan admin
   pertama di luar namespace `learning`; jelaskan dulu apa yang di-apply dan ke mana.
   Jadikan `local-path` StorageClass default.
2. PVC bind → jalankan `pod-with-pvc.yaml`, tulis file ke volume, **hapus Pod, buat lagi,
   buktikan data selamat** (inti "persistent").
3. Konsep: `accessModes` (kenapa RWO paling umum; RWX butuh storage jaringan),
   `volumeBindingMode: WaitForFirstConsumer` vs `Immediate`, `reclaimPolicy` Delete vs Retain
   (di production: data penting → Retain).
4. **Gotcha repo:** `pod-with-pvc.yaml` pakai `nginx:latest` — bahas kenapa tag `latest`
   dilarang di production (tidak reproducible, rollback mustahil dipastikan).

**Checkpoint:** user bisa menceritakan alur PVC-Pending→Bound lengkap dengan komponennya,
dan menjawab: "PVC dihapus, datanya ikut hilang tidak?" (jawaban: tergantung reclaimPolicy).

---

## Modul 4 — Health Check & Self-Healing

**File repo:** `pod/pod-with-probe.yaml` (**sebenarnya Deployment** — bahas kenapa penamaan
file menyesatkan itu masalah nyata di repo tim), `replicaset/replicaset.yaml`.

**Dokumentasi:**
[Liveness/Readiness/Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) ·
[Pod Lifecycle — probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) ·
[ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

### Sesi 4a — Rantai kepemilikan & self-healing
1. Apply `replicaset.yaml` → hapus satu Pod → amati dibuat ulang. Lihat `ownerReferences`.
2. Deployment → ReplicaSet → Pod: kenapa di production selalu Deployment, bukan RS langsung
   (rolling update & rollback hidup di layer Deployment).
3. Bersihkan RS sebelum lanjut (nama & selector bentrok dengan Deployment).

### Sesi 4b — Readiness vs Liveness vs Startup
1. Apply `pod-with-probe.yaml` (readiness aktif). **Sabotase:** ubah path probe ke `/salah`
   → Pod `Running` tapi `0/1 Ready`, endpoint dicabut dari Service, traffic berhenti,
   **container tidak di-restart**. Itulah readiness.
2. Uncomment livenessProbe, sabotase serupa → sekarang container **di-restart** berulang.
   Bandingkan perilaku keduanya di `describe` + events.
3. Parameter timing (`initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`,
   `failureThreshold`) — hitung bersama: berapa lama sampai restart pertama?
4. **Best practice production:** endpoint liveness harus lebih "murah & bodoh" daripada
   readiness (liveness yang ikut cek DB bisa memicu restart massal saat DB lambat = insiden
   klasik); startupProbe untuk aplikasi yang lambat boot, menggantikan `initialDelaySeconds`
   besar.

**Checkpoint:** tanpa melihat catatan, user mengisi tabel: "probe X gagal → apa yang terjadi
pada Pod, Service endpoint, dan restart count". Plus drill: Pod Ready 0/1, temukan sebabnya.

---

## Modul 5 — Resource Management & Tata Kelola Namespace

**File repo:** `pod/pod-with-limit.yaml`, `limitrange/limitrange.yaml`,
`resourcequota/resourcequota.yaml`.

**Dokumentasi:**
[Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) ·
[Pod QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/) ·
[LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) ·
[ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

### Sesi 5a — Requests, limits & QoS
1. `pod-with-limit.yaml`: requests = dasar **scheduling** (dijanjikan), limits = **enforcement**
   runtime. CPU melebihi limit → throttled; memori melebihi limit → **OOMKilled**.
2. Demo OOMKilled: Pod kecil yang sengaja makan memori melebihi limit → `describe` menunjukkan
   `OOMKilled`, restart count naik. (SRE ketemu ini tiap minggu.)
3. QoS class: Guaranteed / Burstable / BestEffort — cek `kubectl get pod -o yaml | grep qosClass`,
   siapa yang dievict duluan saat node tertekan.
4. Best practice production: **requests selalu diisi**; memory limit = memory request (hindari
   overcommit memori); CPU limit itu diperdebatkan (throttling) — kenali dua mazhabnya.

### Sesi 5b — LimitRange & ResourceQuota (governance)
1. Apply `limitrange.yaml` di `learning` → buat Pod **tanpa** `resources` → lihat default
   disuntik otomatis (`describe pod` menunjukkan requests/limits terisi).
2. Apply `resourcequota.yaml` → coba lampaui (scale deployment besar-besaran) → request
   **ditolak saat create** dengan pesan quota. Bahas: quota menolak Pod tanpa requests jika
   quota `requests.*` aktif — makanya LimitRange dan Quota berpasangan.
3. Bingkai production: multi-tenant cluster, satu namespace per tim, quota mencegah satu tim
   menghabiskan cluster.

**Checkpoint:** user bisa menjelaskan kenapa Pod OOMKilled padahal node masih longgar; dan
kenapa Pod baru tertolak quota padahal quota belum penuh (hint: Pod tanpa requests).

---

## Modul 6 — Autoscaling: HPA

**File repo:** `hpa/hpa.yaml` (CPU 70% + memory 75%, min 3 max 6, target Deployment
`nginx-demo`). Prasyarat: metrics-server ✅ sudah ada di RKE2 (`rke2-metrics-server`);
Deployment `nginx-demo` **harus punya `resources.requests`** — hubungkan dengan Modul 5,
karena `averageUtilization` dihitung sebagai % dari requests.

**Dokumentasi:**
[HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) ·
[HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

### Sesi 6
1. Pastikan Deployment `nginx-demo` punya requests (edit dari hasil Modul 5) + Service ClusterIP.
2. Apply `hpa.yaml` → `kubectl get hpa -w`; baca kolom TARGETS.
3. Bangkitkan beban: Pod `busybox` loop `wget` ke Service → amati scale-out ke 6; hentikan
   beban → amati scale-in **lambat** (stabilization window default 5 menit — by design,
   mencegah flapping).
4. Diskusi production: HPA pada CPU untuk web stateless itu baseline; memory-based HPA sering
   menipu (memori jarang turun); custom metrics (RPS/queue) via adapter — konsep saja.
   HPA mengubah `replicas` → jangan set `replicas` manual di manifest yang di-HPA-kan
   (konflik saat re-apply).

**Checkpoint:** user bisa menjelaskan dari mana angka TARGETS dihitung, kenapa HPA butuh
requests, dan kenapa scale-in tidak langsung.

---

## Modul 7 — Expose: Tiga Tipe Service & Ingress

**File repo:** `service/service.yaml` (ClusterIP 8080→80), `service-nodeport.yaml`,
`service-loadbalancer.yaml`, `ingress/ingress.yaml` (host `example.com` → Service
`nginx-demo:8080`). Prasyarat: ingress controller ✅ sudah ada (`rke2-ingress-nginx`).

**Dokumentasi:**
[Service](https://kubernetes.io/docs/concepts/services-networking/service/) ·
[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) ·
[Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

### Sesi 7
1. Recap tiga tipe (sudah kenal NodePort dari Modul 1). Apply `service-loadbalancer.yaml`
   → EXTERNAL-IP **`<pending>` selamanya** di cluster bare-metal — pelajaran penting:
   `LoadBalancer` butuh integrasi cloud/MetalLB.
2. Apply `service.yaml` (ClusterIP) + `ingress.yaml`. Uji routing host-based:
   `curl -H "Host: example.com" http://<NODE_IP>/` — mengerti kenapa header Host yang
   menentukan (L7), tanpa perlu DNS.
3. Tambah rule path (`/app` → service yang sama) untuk merasakan path-based routing +
   `pathType: Prefix` vs `Exact`.
4. Gambar bersama alur lengkap: client → ingress controller (DaemonSet di tiap node) →
   Service ClusterIP → endpoint Pod. Bandingkan dengan NodePort langsung.
5. Konsep production: satu LB + satu ingress controller melayani puluhan host/path
   (vs satu LB per Service); TLS termination di Ingress (`spec.tls` + cert-manager — konsep,
   praktiknya opsional).

**Checkpoint:** user bisa memilih tipe expose yang tepat untuk 3 skenario yang kamu berikan
(internal service-to-service, debugging cepat, production web publik) dengan alasannya.

---

## Modul 8 — Capstone: Mini Production Deployment + Drill Troubleshooting

Tidak ada file repo baru — **user menulis semuanya dari nol** di direktori baru
(mis. `capstone/`), dengan penamaan production (`frontend`, `app-config`, `db-credentials`).

### Sesi 8a — Bangun (user menulis, tutor me-review seperti PR)
Satu aplikasi lengkap di namespace `learning`, memakai semua yang sudah dipelajari:
- Deployment `frontend` (nginx, image di-pin, ≥2 replika) + readiness & liveness probe +
  `resources` lengkap;
- Config dari ConfigMap (file mount) + satu Secret (env);
- PVC untuk direktori data;
- Service ClusterIP + Ingress host-based;
- HPA (setelah semuanya stabil);
- Semua jalan di bawah LimitRange + ResourceQuota namespace.

Review checklist tutor: label konsisten, tidak ada `latest`, requests terisi, probe masuk akal,
tidak ada secret hardcode di manifest yang di-commit.

### Sesi 8b — Drill troubleshooting (ujian akhir)
Tutor merusak satu hal per ronde **tanpa memberi tahu apa**; user mendiagnosis sampai akar
masalah + memperbaiki. Minimal 5 dari daftar:
image typo (`ImagePullBackOff`) · selector Service tidak match (endpoint kosong) · ConfigMap
yang direferensikan dihapus (`CreateContainerConfigError`) · probe salah port (Ready 0/1 /
restart loop) · limit memori terlalu kecil (`OOMKilled`) · quota penuh (create ditolak) ·
PVC unbound (Pod `Pending`) · Ingress ke Service/port yang salah (404/502).
Rujukan alur: [Debug Applications](https://kubernetes.io/docs/tasks/debug/debug-application/).

**Kelulusan fundamental:** semua drill terpecahkan dengan alur diagnosis yang runtut
(gejala → `describe`/`events`/`logs` → akar masalah → fix → verifikasi). Setelah ini user siap
lanjut ke topik di luar cakupan repo (RBAC, NetworkPolicy, Helm, GitOps, observability) —
bahas rencananya saat tiba di sini.
