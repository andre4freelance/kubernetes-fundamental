# Kubernetes Learning — Progress & Recap

Catatan belajar Kubernetes terstruktur, dipakai lintas perangkat (pull repo → buka Claude CLI
→ lanjut dari sini). Panduan tutor + silabus detail ada di skill **`k8s-belajar`** yang ikut
ter-commit di repo (`.claude/skills/k8s-belajar/`) — otomatis terdeteksi Claude Code di mesin
mana pun.

**Tujuan:** siap kerja DevOps/SRE. **Cakupan plan ini:** fundamental (seluruh resource repo
ini) sampai lulus capstone Modul 8; topik lanjutan dibahas setelah itu. **Pace:** fleksibel,
per sesi ±1 jam.

## Silabus (detail: `.claude/skills/k8s-belajar/MODULES.md`)

| Modul | Topik | File repo | Status |
|---|---|---|---|
| 0 | Baseline & orientasi (`get`/`describe`/`logs`/`events`, peta cluster) | — | ✅ |
| 1 | Workload: Deployment, Service, rolling update, ImagePullBackOff | `deployment/`, `service/` | ✅ |
| 2 | Konfigurasi: ConfigMap & Secret (env vs mount, base64 ≠ enkripsi, drill `CreateContainerConfigError`) | `configmap/`, `secret/`, `deployment/deployment-{configmap,secret}-*`, `pod/pod-with-{cm,secret}` | 🔄 Sesi 2a ✅ |
| 3 | Storage: PVC/PV/StorageClass (pasang local-path-provisioner, bukti data persisten) | `pvc/`, `pod/pod-with-pvc` | ⬜ |
| 4 | Health check & self-healing: ReplicaSet ownership, readiness vs liveness vs startup | `replicaset/`, `pod/pod-with-probe` | ⬜ |
| 5 | Resource management: requests/limits, QoS, OOMKilled, LimitRange, ResourceQuota | `pod/pod-with-limit`, `limitrange/`, `resourcequota/` | ⬜ |
| 6 | Autoscaling: HPA + load test (metrics-server sudah ada di RKE2) | `hpa/` | ⬜ |
| 7 | Expose: 3 tipe Service + Ingress host/path routing (ingress-nginx sudah ada) | `service/`, `ingress/` | ⬜ |
| 8 | **Capstone:** deploy production-style dari nol + drill troubleshooting (ujian akhir) | `capstone/` (ditulis user) | ⬜ |

## Setup

- **Cluster:** RKE2/Rancher, 1 master + 1 worker.
- **Namespace kerja:** `learning` (semua latihan pakai `-n learning`).
- **Kubeconfig:** file lokal per-perangkat (context `local`). Tidak disimpan di repo.
- **Konvensi penamaan:** pakai gaya production berbahasa Inggris (mis. `frontend`, `api`,
  `app-config`, `db-credentials`) supaya terbiasa dengan dunia nyata.

## Progress

### ✅ Modul 0 — Baseline & Orientasi
- Beda & kapan pakai `get` / `describe` / `logs` / `get events`.
  - `get`=status ringkas · `describe`=detail+events objek · `logs`=output aplikasi di container ·
    `events`=riwayat scheduler/kubelet se-namespace.
  - Pod `Running` tapi app error → pakai **`logs`** (bukan `describe`).
- Memetakan cluster: node & role, versi server, namespace sistem (`kube-system`, `cattle-*`,
  `fleet-*`, `cert-manager`) vs namespace aman (`learning`). Jangan sentuh namespace sistem.

### ✅ Modul 1 — Workload Fundamental
- **Deployment + Service:** `create deployment` (3 replika) + `expose` NodePort. Akses eksternal
  via `NODE_PUBLIC_IP:NODEPORT`. Range NodePort default 30000–32767.
- **Service = load balancer L4:** kube-proxy (iptables/IPVS) menyebar koneksi ke endpoint Pod.
  `type: LoadBalancer` = LB eksternal cloud (beda hal); `Ingress` = L7 (host/path). Balancing
  Service itu per-koneksi, bukan per-request.
- **Kenapa Service perlu walau Pod punya IP:** IP Pod fana (berubah saat Pod di-recreate) & ada
  banyak replika → Service kasih 1 alamat stabil + load balancing.
- **Scheduling & taint:** Pod tersebar antar-node. Node master di cluster ini **tanpa taint**
  `NoSchedule`, jadi ikut menerima Pod (dibiarkan schedulable demi kapasitas). Tiap node punya
  pod-CIDR sendiri (master `10.42.2.0/24`, worker `10.42.1.0/24`).
- **Rolling update & rollback:** `set image` mengganti Pod bertahap (Pod baru Ready dulu, baru
  yang lama mati → tanpa downtime), diatur `maxSurge`/`maxUnavailable` (default 25%/25%).
  `rollout history` + `rollout undo` (rollback menghidupkan lagi ReplicaSet lama).
- **Diagnosa ImagePullBackOff:** status ini **gejala**, bukan penyebab. Baca **Events** di
  `describe` untuk tahu sebab: `not found`=nama/tag salah · `i/o timeout`/`dial tcp`=jaringan ·
  `unauthorized`=registry privat · `toomanyrequests`=rate limit. `ErrImagePull`=gagal pull;
  `ImagePullBackOff`=sudah retry berulang dengan backoff (jeda makin lama).

### 🔄 Modul 2 — Konfigurasi (ConfigMap & Secret)

**Sesi 2a ✅ — ConfigMap (env & file mount):**
- **Reconciliation loop = konsep inti.** Apply Deployment *sebelum* ConfigMap-nya ada:
  `apply` tetap diterima (API server cuma validasi skema, tidak cek referensi), Pod
  `CreateContainerConfigError` — kubelet yang me-resolve referensi saat create container dan
  **retry terus** (`x9 over 102s` di Events). Begitu ConfigMap di-apply, Pod sembuh sendiri:
  nama Pod sama, RESTARTS 0. Urutan apply tidak penting; sistemnya eventually consistent.
- **Anatomi manifest:** 4 bagian universal (`apiVersion`/`kind`/`metadata`/`spec`) + `status`
  (diisi cluster). Deployment = matryoshka: `spec.template.spec` adalah spec milik Pod.
  Label = satu-satunya lem antar objek (selector Deployment & Service).
- **Aturan mutlak rollout:** perubahan apa pun di dalam `template` → pod-template-hash baru →
  ReplicaSet baru → rolling update (dibuktikan: ganti `envFrom` saja hash berubah — env dibaca
  proses hanya saat start, jadi wajib Pod baru). `replicas` di luar template → scaling saja,
  Pod lama tak disentuh (dibuktikan tanpa sengaja: apply image & replicas dua tahap → umur Pod
  terbelah 5m39s vs 34s, hash sama).
- **`env`+`configMapKeyRef` vs `envFrom`:** selektif + bisa rename vs sedot semua key
  (nama env = nama key); `env` menang saat tabrakan. Dibuktikan: `grep APP` 1 baris → 3 baris.
- **ConfigMap sebagai file:** pasangan `volumes` (level Pod, *apa*) + `volumeMounts` (level
  container, *di mana*); key ConfigMap jadi nama file; mount **menimpa** isi direktori asal.
  Halaman nginx diganti murni dari YAML, diakses via Service NodePort yang ditulis dari nol
  (`service/service-demo-app.yaml`) — plus kebiasaan `kubectl get endpoints` untuk cek
  selector nyambung (catatan: Endpoints deprecated v1.33+ → `get endpointslices`).
- **Update live:** ubah ConfigMap → env var TIDAK berubah selamanya; file mount berubah
  *eventually* (±30–60 dtk, sinkronisasi periodik kubelet) tanpa restart — dibuktikan dengan
  2x curl. Production: tetap `rollout restart` setelah ubah config biar deterministik.
- **Luka latihan (yang bikin ingat):** uncomment YAML meninggalkan 1 spasi ekstra →
  `did not find expected key` (obat: `kubectl apply --dry-run=client`); zsh menelan `[0]`
  tanpa kutip; `exec` ke Pod yang salah saat verifikasi; env `APP_VERSION=1.25` padahal
  container 1.26 — config bisa menyimpang diam-diam dari realita.

**⏭️ Berikutnya — Sesi 2b: Secret** (`data` vs `stringData`, base64 ≠ enkripsi, mount
credential file, drill `deployment-secret-1` yang mereferensikan Secret yang tidak ada).
Warm-up ulang: jawab dulu — di `demo-app`, kenapa `ls /usr/share/nginx/html` hanya
menampilkan `index.html` tanpa `50x.html`? (Belum sempat diverifikasi di sesi 2a.)
Keadaan cluster saat ini: Deployment `nginx-demo` (5 replika, nginx:1.26, envFrom aktif),
Pod `demo-app` + Service NodePort `demo-app`, Deployment `web` (sisa Modul 1) — bersihkan
`nginx-demo` sebelum apply workload `nginx-demo` lain.
(Catatan silabus: Storage/PVC jadi Modul 3 — cluster belum punya StorageClass, itu bagian
pelajarannya.)

## Cara melanjutkan di perangkat lain
1. `git pull` repo ini — skill tutor + silabus ikut terbawa (`.claude/skills/k8s-belajar/`).
2. Pastikan kubeconfig cluster `learning` tersedia di perangkat itu (context `local`);
   path-nya per-mesin dan tidak disimpan di repo.
3. Buka Claude Code di repo ini, minta **"lanjut belajar Kubernetes"** — skill `k8s-belajar`
   terdeteksi otomatis, membaca file ini, dan melanjutkan dari checkpoint terakhir.
