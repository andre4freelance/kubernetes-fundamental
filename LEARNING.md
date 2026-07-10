# Kubernetes Learning — Progress & Recap

Catatan belajar Kubernetes terstruktur, dipakai lintas perangkat **dan lintas AI tool**
(pull repo → buka AI assistant apa pun → lanjut dari sini). Instruksi tutor untuk AI ada di
**`AGENTS.md`** di root repo (standar lintas-tool); untuk Claude Code ada padanannya berupa
skill **`k8s-belajar`** (`.claude/skills/k8s-belajar/`) yang terdeteksi otomatis. Silabus
detail per modul: `.claude/skills/k8s-belajar/MODULES.md`.

**Tujuan:** siap kerja DevOps/SRE. **Cakupan plan ini:** fundamental (seluruh resource repo
ini) sampai lulus capstone Modul 8; topik lanjutan dibahas setelah itu. **Pace:** fleksibel,
per sesi ±1 jam.

## Silabus (detail: `.claude/skills/k8s-belajar/MODULES.md`)

| Modul | Topik | File repo | Status |
|---|---|---|---|
| 0 | Baseline & orientasi (`get`/`describe`/`logs`/`events`, peta cluster) | — | ✅ |
| 1 | Workload: Deployment, Service, rolling update, ImagePullBackOff | `deployment/`, `service/` | ✅ |
| 2 | Konfigurasi: ConfigMap & Secret (env vs mount, base64 ≠ enkripsi, drill `CreateContainerConfigError`) | `configmap/`, `secret/`, `deployment/deployment-{configmap,secret}-*`, `pod/pod-with-{cm,secret}` | ✅ |
| 3 | Storage: PVC/PV/StorageClass (pasang local-path-provisioner, bukti data persisten) | `pvc/`, `pod/pod-with-pvc` | ✅ |
| 4 | Health check & self-healing: ReplicaSet ownership, readiness vs liveness vs startup | `replicaset/`, `pod/pod-with-probe` | ✅ |
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

### ✅ Modul 2 — Konfigurasi (ConfigMap & Secret) — LULUS checkpoint

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

**Sesi 2b ✅ — Secret (+ checkpoint Modul 2 LULUS):**
- **Mount shadowing** (PR dari 2a): volume mount **menutupi** isi direktori asal (analogi
  USB drive) — `50x.html` tidak dihapus, cuma tertutup mount point. Solusi tambah-satu-file:
  `subPath` — tapi file subPath **tidak ikut live-update** saat ConfigMap berubah (trade-off).
- **base64 = encoding, bukan enkripsi** — dibongkar 3 lapis: (1) `base64 -d` sedetik;
  (2) `stringData` vs `data` tersimpan identik di cluster; (3) plaintext malah bocor utuh di
  annotation `last-applied-configuration`. Yang membatasi akses Secret itu **RBAC**, bukan
  `kind`-nya; plus at-rest encryption di etcd (RKE2: default on) & mount via tmpfs.
  Jangan pernah commit Secret bernilai asli ke git.
- **Secret di container selalu sudah ter-decode** (env maupun file mount) — base64 cuma
  kemasan transportasi di lapisan API.
- **Pod itu (hampir) immutable** — apply perubahan volumes/env ke bare Pod ditolak
  (`Forbidden`); hanya `image` dkk yang boleh. "Perubahan" di k8s = **penggantian** Pod;
  Deployment kelihatan mutable karena controller yang mengganti Pod untukmu. Bare Pod =
  kamu controller manualnya (delete + apply).
- **Insiden label-selector** (dialami langsung): Pod `demo-app` baru tanpa label → Service
  endpoints kosong → `curl: connection reset` **padahal semua komponen "sehat"**. Refleks:
  `kubectl get endpoints` (deprecated v1.33+ → `get endpointslices`). Tiket insiden paling
  umum se-k8s.
- **DRILL #1 LULUS:** `deployment-secret-1` → `CreateContainerConfigError` → akar: referensi
  Secret `app-secret` tidak ada → fix: buat `secret/app-secret.yaml` (Solusi B, blast radius
  kecil vs Solusi A ubah referensi). Pelajaran incident: gejala ≠ akar masalah (gejala =
  status yang terlihat); **perubahan minimal saat fix insiden** (jangan sekalian ganti
  env→envFrom); `logs` kosong untuk container yang tak pernah start — `describe`/Events dulu.

### ✅ Modul 3 — Storage (PVC/PV/StorageClass) — LULUS checkpoint

**Pemanasan — tabrakan label & sifat objek:**
- **Deployment & Service itu sejajar**, bukan hierarki. Dua relasi berbeda: *kepemilikan*
  (vertikal, `ownerReferences`: Deployment→ReplicaSet→Pod — hapus induk, anak tersapu
  = cascading deletion; `--cascade=orphan` menyisakan anak yatim) vs *selektor* (horizontal,
  lewat label: Service→Pod, HPA→Deployment, Ingress→Service — putus satu, yang lain tetap
  hidup jadi *dangling*). Dibuktikan: hapus 3 Deployment → semua Pod ikut mati, tapi Service
  `demo-app`/`web` tetap ada karena tak dimiliki Deployment mana pun.
- **Bahaya label kembar:** `nginx-demo` & `nginx-deploy` sama-sama `app=nginx` → satu Service
  `selector: app=nginx` akan menyedot **8 Pod campur aduk** (beda image/config) → respons tak
  konsisten walau semua "Running". Konvensi `app.kubernetes.io/{name,instance}` ada buat cegah ini.

**Inti — 3 objek abstraksi & dynamic provisioning:**
- **PV** (storage nyata, cluster-scoped) ← **PVC** (permintaan, *namespaced* — Pod cuma bisa
  mount PVC senamespace) ← **StorageClass** (pabrik yang bikin PV otomatis, cluster-scoped).
  Pod cuma kenal `claimName` PVC — tak pernah sebut Ceph/EBS/local-path. Itulah lapisan portable.
- **Cluster ini RKE2 self-managed di VM GCP → tak ada StorageClass bawaan** (beda dg GKE yang
  punya `standard-rwo` via CSI). Maka `pvc/pvc.yaml` → `Pending`. Events gamblang:
  `no persistent volumes available ... and no storage class is set`. Diagnosa PVC = `describe`,
  baca Events (refleks sama seperti ImagePullBackOff).
- Pasang **local-path-provisioner** (disimpan ke `pvc/local-path-provisioner.yaml`). Dua kejutan:
  (1) StorageClass `local-path` **bukan default** → PVC dengan `storageClassName` nil tetap
  Pending (minta "default", tak ada yang default). Fix best-practice: sebut kelas **eksplisit**
  `storageClassName: local-path` (explicit > implicit; `storageClassName` immutable → **delete +
  recreate** PVC). (2) `volumeBindingMode: WaitForFirstConsumer` → PV **baru lahir saat Pod
  pertama memakai** PVC (`waiting for first consumer`), bukan langsung.
- **Persistensi dibuktikan:** tulis `bukti.txt` ke volume → `delete pod` → `apply` ulang → file
  **selamat**, RESTARTS 0 (Pod benar-benar baru). Siklus hidup data ≠ siklus hidup Pod.
- **Mekanisme node-lokal:** local-path **tidak** replikasi antar-node; data duduk di
  `/opt/local-path-provisioner` di **satu** node. PV dipaku ke node itu via `nodeAffinity`
  (`kubernetes.io/hostname In [rke-worker]`) → Pod baru **diseret balik** ke node yang sama.
  Bukan data mengejar Pod; Pod-nya yang ditarik ke data. **Batasan:** node mati = data terkubur
  → production butuh storage jaringan node-independent (Ceph/GCP PD via CSI/EBS).
- **reclaimPolicy `Delete`** (bawaan local-path): hapus PVC → PV + data ikut hilang.
  `Retain` = data diselamatkan buat recovery manual (pilih ini untuk data penting).
- **DRILL #2 LULUS:** Pod `broken-storage` → `claimName` PVC tak ada → **Pending**
  (`FailedScheduling`, akar: `persistentvolumeclaim "storage-yang-salah" not found`). Nuansa:
  PVC RWO cuma bisa dibagi 2 Pod kalau **se-node** — "perbaikan" harus sadar access mode + node.
- **Luka kecil:** `pod/pod-with-pvc.yaml` pakai `nginx:latest` (anti-pattern: deploy
  non-reproducible, restart bulan depan bisa dapat versi beda, rollback tak bermakna → selalu pin).

### ✅ Modul 4 — Health Check & Self-Healing — LULUS checkpoint

**Setup lintas-laptop (bukan k8s, tapi bikin nyangkut):** di laptop kerja default
`~/.kube/config` cuma berisi `current-context: rancher` TANPA definisi context → `kubectl` error
"context was not found". Fix: **merge** kubeconfig latihan ke default —
`KUBECONFIG=~/.kube/rke.config:~/.kube/config kubectl config view --flatten > config.merged` →
timpa `~/.kube/config`. Sekarang default punya 2 context: **`local`** (akses langsung RKE2) &
**`rancher`** (lewat Rancher proxy — ternyata cluster yang SAMA). Pindah cukup `use-context`,
tak perlu `export KUBECONFIG`. Refleks tetap: `current-context` = `local` sebelum mutate.

**Self-healing (ReplicaSet) — Sesi 4a:**
- **Reconciliation loop lagi:** RS controller terus mencocokkan `desired` vs `actual`. Hapus 1
  dari 3 Pod → actual jadi 2 → controller **bikin Pod BARU** (nama baru, umur 0) menutup gap.
  Bukan "menghidupkan Pod lama" — itu penggantian. Nama Pod RS = `nginx-demo-<random>`
  (Deployment nambah segmen hash: `nginx-demo-<hash>-<random>`).
- **Kepemilikan & cascading deletion:** `ownerReferences` di Pod = "akta kelahiran" (muncul di
  `describe` sbagai `Controlled By: ReplicaSet/...`). **Garbage collector** menyapu anak saat
  induk dihapus. `kubectl delete rs` (default) → 3 Pod ikut mati. `--cascade=orphan` → RS hilang,
  3 Pod **yatim tetap hidup** (dibuktikan langsung). Lanjutan Modul 3 (Deployment→RS→Pod).
- **3 policy propagasi:** `background` (default — induk mati dulu, anak nyusul async),
  `foreground` (anak habis dulu, induk baru resmi hilang — induk nyangkut `Terminating`),
  `orphan` (putus `ownerReferences`, anak dibiarkan). Field API: `propagationPolicy`.

**Probes — Sesi 4b (inti modul):**
- **Tiga probe = tiga pertanyaan berbeda.** readiness "siap terima traffic?", liveness "masih
  hidup/tidak nge-hang?", startup "sudah selesai booting?".
- **DRILL readiness (`httpGet` path → `/salah`, 404):** Pod tetap **`Running` tapi `0/1`**,
  **RESTARTS 0** (readiness TIDAK membunuh), dicopot dari routing. Nuansa EndpointSlice (vs
  `Endpoints` lama): alamat Pod sakit **tetap terdaftar** tapi dgn `conditions.ready:false` →
  kube-proxy skip. **Efek rollout:** ubah probe = ubah `template` → RS baru → tapi Pod baru tak
  pernah Ready → dengan `maxUnavailable` default, Deployment **menolak membunuh Pod lama sehat**
  → **rollout MACET**. Pelajaran: readiness probe yang benar = **gerbang keamanan deploy**
  (versi rusak tak menggantikan versi sehat). Readiness punya 2 peran: gerbang traffic runtime +
  gerbang rollout.
- **DRILL liveness (path → `/salah`):** RESTARTS **naik terus**, status **`CrashLoopBackOff`**,
  container **dibunuh & di-restart di tempat** (nama Pod TETAP — beda dari readiness). `BackOff`
  = jeda restart eksponensial (10→20→40s… cap 5 mnt), konsep sama `ImagePullBackOff` Modul 1.
  Timing restart pertama: `initialDelaySeconds 10` + `failureThreshold 3 × periodSeconds 10` ≈
  **30 dtk**.
- **Insight production terpenting:** nginx-nya SEHAT — yang membunuh adalah **probe kita yang
  salah**. → **`CrashLoopBackOff` ≠ selalu "app crash"**; bisa "liveness terus membunuh app
  sehat". Tiket insiden klasik: **liveness yang ikut cek DB** → DB lambat → **semua** Pod gagal
  liveness **serentak** → **restart massal** → outage kecil jadi besar. Aturan: **liveness harus
  lebih murah & bodoh dari readiness** (cek dependency itu tugas readiness — efeknya reversibel).
- **startupProbe** (nalar, tak hands-on): gerbang sebelum liveness/readiness — selama belum lolos,
  keduanya **ditahan**, biar app yang lambat boot (mis. 90 dtk load model) tak dibunuh liveness di
  tengah boot. Gagal terus → berperilaku **seperti liveness** (restart). Pengganti trik lama
  `initialDelaySeconds` besar (yang jelek: delay itu berlaku tiap restart, bukan cuma boot pertama).
- **Tabel kunci (analogi kasir toko — readiness=saklar traffic reversibel; liveness/startup=tombol
  restart yang membunuh):**

  | Probe gagal | Pod STATUS | Dapat traffic? | RESTARTS |
  |---|---|---|---|
  | readiness | `Running` | ❌ | `0` |
  | liveness | `CrashLoopBackOff` | ❌ | 🔺 naik |
  | startup (terus) | `CrashLoopBackOff` | ❌ | 🔺 naik |

- **Cara sabotase deklaratif:** user mengedit `pod/pod-with-probe.yaml` sendiri (uncomment
  `livenessProbe`, path `/salah`) lalu `apply` — bukan `kubectl patch` imperatif. Bahas imperative
  (`patch`/`edit`/`scale` — cepat, jejak hilang, buat hotfix/CI) vs declarative (`apply -f` — di
  git, bisa review/rollback, norma sehari-hari). Refleks: `apply --dry-run=client` sebelum apply.

**⏭️ Berikutnya — Modul 5: Resource Management & Governance** (`pod/pod-with-limit`, `limitrange/`,
`resourcequota/`). Target: requests (dasar scheduling) vs limits (enforcement runtime), demo
**OOMKilled** (memori > limit), QoS class (Guaranteed/Burstable/BestEffort — siapa dievict duluan),
lalu LimitRange (default resource disuntik otomatis) + ResourceQuota (create ditolak saat lampaui
quota; kenapa Quota `requests.*` butuh LimitRange). Nyambung ke Modul 6 (HPA butuh requests).

## Keadaan cluster saat ini (verifikasi di awal sesi berikutnya)

- **StorageClass `local-path` terpasang permanen** (manifest: `pvc/local-path-provisioner.yaml`),
  sengaja **non-default** — PVC harus eksplisit `storageClassName: local-path`.
- **Sisa drill Modul 4 kemungkinan masih jalan** di namespace `learning`: Deployment+Service
  `nginx-demo` (readiness `/` sehat + liveness `/salah` rusak → `CrashLoopBackOff`).
  **Bersihkan dulu sebelum mulai Modul 5** (`kubectl -n learning delete -f pod/pod-with-probe.yaml`
  atau delete deployment/service `nginx-demo`).
- `pod/pod-with-probe.yaml` di repo **sudah di-revert** ke kondisi asli (liveness ter-comment) —
  tidak ada aksi git yang tertunda.

## Cara melanjutkan di perangkat lain
1. `git pull` repo ini — instruksi tutor ikut terbawa: `AGENTS.md` (semua AI tool) +
   skill `k8s-belajar` (`.claude/skills/k8s-belajar/`, khusus Claude Code).
2. Pastikan kubeconfig cluster `learning` tersedia di perangkat itu (context `local`);
   path-nya per-mesin dan tidak disimpan di repo.
3. Buka AI tool pilihan di repo ini, minta **"lanjut belajar Kubernetes"**:
   - **Claude Code** → skill `k8s-belajar` terdeteksi otomatis;
   - **tool lain** (Codex, Gemini CLI, Cursor, dll.) → tool membaca `AGENTS.md`.
   Keduanya membaca file ini dan melanjutkan dari checkpoint ⏭️ terakhir.
