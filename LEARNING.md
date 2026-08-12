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
| 5 | Resource management: requests/limits, QoS, OOMKilled, LimitRange, ResourceQuota | `pod/pod-with-limit`, `limitrange/`, `resourcequota/` | ✅ |
| 6 | Autoscaling: HPA + load test (metrics-server sudah ada di RKE2) | `hpa/` | ✅ |
| 7 | Expose: 3 tipe Service + Ingress host/path routing (ingress-nginx sudah ada) | `service/`, `ingress/` | ⬜ |
| 8 | **Capstone:** deploy production-style dari nol + drill troubleshooting (ujian akhir) | `capstone/` (ditulis user) | ⬜ |

## Setup

- **Cluster: fleksibel per sesi** — repo ini tidak terikat ke satu cluster. Di awal tiap sesi
  user menyebut cluster latihan yang dipakai, dan AI menyinkronkan pilihan itu ke config MCP
  semua tool (protokol lengkap: `AGENTS.md` / `.claude/skills/k8s-belajar/SKILL.md`).
  Cluster yang pernah dipakai (referensi, bukan default):
  - **RKE2/Rancher** (1 master + 1 worker) — context `local`;
  - **kubeadm cross-cloud lab** (1 master + 2 worker, Alibaba + Azure) — context `kubeadm-lab`.
- **Namespace kerja:** `learning` (semua latihan pakai `-n learning`), sama di semua cluster.
- **Kubeconfig:** file lokal per-perangkat, **satu file terisolasi per cluster** — tidak pernah
  disimpan di repo, tidak pernah digabung ke `~/.kube/config` default (itu cluster production).
- **Komponen cluster (StorageClass, metrics-server, ingress controller) itu per-cluster** —
  status terpasangnya dicatat di bagian "Keadaan cluster saat ini" per cluster, jangan
  diasumsikan ada hanya karena pernah dipasang di cluster lain.
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

### ✅ Modul 5 — Resource Management & Tata Kelola Namespace — LULUS checkpoint

Dikerjakan di cluster **kubeadm-lab** (bukan RKE2) — sesi ini yang pertama kali cluster latihan
dipilih eksplisit di awal sesi (lihat protokol baru di `AGENTS.md`/`.claude/skills/k8s-belajar/`:
tanya cluster tiap sesi, sync ke `.mcp.json`/`opencode.json`, jangan pernah asumsikan).

**Sesi 5a — Requests, limits & QoS:**
- **Unit CPU `m` (millicpu):** `1000m` = 1 core penuh; ini jatah *waktu* CPU (cgroup CFS quota
  per periode), bukan pecahan core fisik.
- **Kenapa CPU di-throttle tapi memory di-`OOMKilled`:** CPU = resource *compressible* (soal
  waktu, bisa "diperas"/dijadwalkan lebih jarang, proses tetap hidup). Memory = *incompressible*
  (soal ruang byte yang sudah dialokasikan — nggak bisa "diperlambat", begitu cgroup limit
  dilampaui, kernel cuma punya opsi bunuh proses via OOM killer).
- **QoS class — syarat presisi** ([docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)):
  `Guaranteed` = *setiap* container punya requests+limits CPU & memory **DAN requests==limits
  persis**; `Burstable` = ada requests/limits tapi nggak memenuhi syarat Guaranteed (mis. requests
  100m/128Mi ≠ limits 200m/256Mi — nama "Burstable" pas: dijamin dapat requests, boleh "meledak"
  sampai limits kalau node longgar); `BestEffort` = tanpa requests/limits sama sekali, bisa makan
  resource sepuasnya tapi juga dikorbankan duluan. Urutan eviction saat node tertekan:
  **BestEffort → Burstable → Guaranteed** (Guaranteed nyaris nggak pernah, karena nggak ada
  bagian yang "melebihi request" — requests-nya = limits-nya).
- **DRILL OOMKilled LULUS:** Pod `pod-oom-demo.yaml` (image `polinux/stress`, `limits.memory:
  256Mi`, `args: --vm-bytes 300M`) → `Reason: OOMKilled`, `Exit Code: 137` (=`128+9`, sinyal
  `SIGKILL` — beda dari `exit code 1` biasa). Insight inti (checkpoint): **cgroup memory limit
  itu lokal ke container**, node yang longgar RAM-nya sama sekali nggak relevan — begitu proses
  di dalam container coba alokasi melebihi limit cgroup-nya sendiri, OOM killer level-cgroup
  langsung turun tangan.
  - **Luka latihan (yang bikin ingat):** placeholder `<angka-lebih-besar-dari-limit>` di skeleton
    contoh ke-apply **secara literal** sebagai args → `stress` gagal parsing → `exit code 1`
    (`reason: Error`, BUKAN OOMKilled) → `CrashLoopBackOff` palsu. Diagnosa lewat
    `lastState.terminated.reason` + `exitCode` (1 vs 137) buat bedain "app error biasa" vs
    "beneran dibunuh OOM" sebelum nebak-nebak.
- **Production framing:** `memory limit == memory request` best practice (beda angka →
  node bisa overcommit kalau banyak Pod burst bareng → OOM level-node yang lebih parah, kena
  Pod lain juga). CPU limit **diperdebatkan**: sebagian tim sengaja nggak pasang (biar nggak
  throttling-induced latency saat node longgar), sebagian tetap pasang (predictability, cegah
  noisy neighbor) — pertanyaan interview SRE klasik.

**Sesi 5b — LimitRange & ResourceQuota (governance):**
- **Dua alasan Pod ditolak quota — beda pesan, beda akar masalah** (checkpoint inti):
  1. **Belum dideskripsikan** (`ResourceQuota` nge-track `requests.*`/`limits.*`, Pod nggak
     nyebut sama sekali) → ditolak **walau kuota masih longgar**: `must specify limits.cpu ...
     requests.memory ...`. Dibuktikan: `test-noreq` ditolak dengan `requests.memory: 4Gi` masih
     jauh dari penuh.
  2. **Beneran lampaui hard cap** → `exceeded quota: ..., requested: pods=1, used: pods=20,
     limited: pods=20`. Dibuktikan: scale `quota-test` ke 20 replika di namespace yang sudah
     isi 9 Pod → cuma 11 yang jadi (9+11=20), sisanya ditolak di level ReplicaSet
     (`FailedCreate` event), Deployment sendiri nggak error, cuma nyangkut `11/20` — reconciliation
     loop (Modul 4) akan otomatis lanjut kalau kuota dilonggarkan, tanpa aksi manual.
- **Kenapa LimitRange menghilangkan penolakan #1:** admission control jalan berurut —
  **LimitRange (mutating, nyuntik default) jalan DULU, baru ResourceQuota (validating, cuma
  memeriksa) jalan**. Begitu LimitRange aktif, Pod kosong sampai ke tahap quota-check sudah
  lengkap ke-4 field-nya (bukti: annotation `kubernetes.io/limit-ranger: LimitRanger plugin
  set: ...`) → quota nggak punya alasan nolak lagi. LimitRange cuma ngisi field yang **kosong**,
  nggak nimpa yang sudah diisi manual.
- **Namespace-scoped, bukan node-scoped** (koreksi kecil sesi ini) — `LimitRange`/`ResourceQuota`
  itu governance per-namespace/tim, nggak nyentuh kapasitas node sama sekali. Best practice
  production: **satu namespace tim = LimitRange + ResourceQuota sepasang** (Quota doang bikin
  developer frustrasi lupa isi resources; LimitRange doang nggak nyegah satu tim habisin cluster).

### ✅ Modul 6 — Autoscaling (HPA) — LULUS checkpoint

Dikerjakan di cluster **kubeadm-lab**, 2026-08-12.

**Prasyarat — metrics-server tidak bawaan di kubeadm-lab (beda dari RKE2):**
- Install manifest resmi → Pod `Running` tapi `0/1`, `kubectl top nodes` gagal. Log:
  `x509: cannot validate certificate for <IP> because it doesn't contain any IP SANs`.
- **Sama pola dengan insiden SAN apiserver session sebelumnya, tapi beda sertifikat:** ini
  kubelet serving cert (self-signed per-node, default cuma hostname SAN), bukan apiserver cert
  — jadi fix `kubeadm init phase certs apiserver` **tidak berlaku** di sini (tidak ada satu cert
  terpusat untuk diregenerate, tiap node punya sendiri). Fix yang dipakai: patch args
  `--kubelet-insecure-tls` ke Deployment `metrics-server` (cuma melemahkan verifikasi TLS
  metrics-server→kubelet buat scrape metrics, bukan auth/otorisasi apapun — beda dari
  `insecure-skip-tls-verify` di client kubectl yang dulu sengaja dihindari). Detail lengkap +
  command: `knowledge/runbooks/kubernetes-cross-cloud-kubeadm-flannel-over-tunnel.md`.

**Inti — cara kerja HPA:**
- **Rumus `TARGETS`:** `averageUtilization% = (rata-rata usage aktual dari metrics-server) ÷
  (resources.requests) × 100`. Requests itu penyebut tetap; usage aktual itu pembilang yang
  bergerak. Makanya HPA **wajib** Deployment target punya `resources.requests` (nyambung Modul 5).
- **DRILL bangkitkan beban — 2 kegagalan berturut sebelum berhasil, masing-masing pelajaran:**
  1. Luka YAML lagi: `resources:` ditulis sejajar dengan `- name: nginx` (item list container),
     bukan sejajar `image`/`ports` (field di dalam item container) → `did not find expected '-'
     indicator`. Aturan: semua field satu container harus sejajar satu sama lain, satu level
     lebih dalam dari key `containers:` yang menaunginya.
  2. 1 Pod busybox loop `wget` sekuensial tanpa jeda → CPU mentok ~6-7%, tidak naik. Sebab:
     `wget` sekuensial = satu request tunggu selesai baru request berikutnya; Service
     load-balance **per-koneksi** (Modul 1) membagi beban kecil itu rata ke 3 Pod → tiap Pod
     nyaris nganggur. Fix: banyak **proses** paralel (loop di-background, `&`) **dan** banyak
     **Pod** generator sekaligus (1 Pod paralel mentok ~24% karena client-nya sendiri jadi
     bottleneck) — baru dengan 3 Pod × 30 loop paralel CPU tembus 70%+.
- **Formula scale-out dibuktikan langsung:** `desiredReplicas = ceil(currentReplicas ×
  currentMetric/target)`. Replicas 3, CPU terbaca ~73% saat scale terjadi, target 70% →
  `ceil(3 × 73/70) = ceil(3.13) = 4` — persis yang terjadi (bukan langsung lompat ke `maxReplicas`
  6). Setelah naik ke 4, beban total yang sama terbagi ke Pod lebih banyak → % per Pod turun →
  cukup stabil di 4, tidak lanjut naik (karena generator tidak ditambah lagi).
- **Scale-out cepat, scale-in sengaja lambat:** begitu beban dihapus, `TARGETS` turun ke 0%
  dalam hitungan detik, tapi `REPLICAS` bertahan di 4 selama **~5 menit** (stabilization window
  default) sebelum turun ke `minReplicas` 3 — HPA pakai nilai **tertinggi** dalam window 5 menit
  terakhir untuk keputusan scale-in, bukan nilai sesaat. Alasan production: mencegah *flapping*
  (Pod baru butuh waktu Ready — kalau keburu di-scale-in lalu naik lagi, user kena latency/error
  saat trafik naik-turun cepat, mis. jam sibuk).
- **Konflik `replicas:` Deployment vs HPA:** keduanya menulis ke field **yang sama persis**
  (`Deployment.spec.replicas`) lewat jalur berbeda — `kubectl apply` menulis langsung, HPA
  menulis lewat **Scale subresource** tiap sync (~15 detik). Re-apply manifest yang masih
  hardcode `replicas: 3` sambil HPA aktif → replicas sempat balik ke 3, lalu ditarik lagi oleh
  HPA di siklus berikutnya (dibuktikan minta user coba live). `minReplicas`/`maxReplicas` HPA
  cuma batas atas-bawah, bukan yang "menimpa" — nilai aktualnya hasil kalkulasi formula di atas,
  bisa berapa saja di antara keduanya (kasus ini: 4). Best practice: **hapus field `replicas:`
  dari manifest Deployment yang sudah di-HPA-kan**, biar HPA satu-satunya penulis field itu —
  belum diterapkan ke `deployment/deployment.yaml` di repo ini (masih `replicas: 3` tertulis),
  jadi kalau file itu di-reapply nanti selagi HPA aktif, tarik-tambangnya akan terulang.

**Checkpoint:** user menjelaskan rumus `TARGETS`, kenapa HPA butuh `requests`, dan kenapa
scale-in tidak langsung — lulus dengan 2 koreksi kecil (rumus butuh usage aktual bukan cuma
requests; `min`/`max` itu batas bukan penimpa langsung).

**⏭️ Berikutnya — Modul 7: Expose (3 tipe Service + Ingress)** (`service/`, `ingress/`).
Prasyarat: ingress controller — verifikasi dulu ada/tidak di cluster yang dipakai sesi itu
(`kubectl get pods -A | grep -i ingress`); di kubeadm-lab kemungkinan besar **belum ada**
(pola sama seperti metrics-server tadi, cluster ini bare-metal self-managed) — pasang manual
kalau perlu (`ingress-nginx`, manifest bare-metal/NodePort).

## Keadaan cluster saat ini (verifikasi di awal sesi berikutnya)

Cluster latihan itu **fleksibel per sesi** sejak 2026-08-11 (lihat protokol di `AGENTS.md`/
`.claude/skills/k8s-belajar/`) — state di bawah dipisah per cluster, jangan diasumsikan cluster
yang sama dipakai lagi.

**Cluster `kubeadm-lab`** (context `kubeadm-lab`, dipakai sesi Modul 6, 2026-08-12) — namespace
`learning` **bersih** di akhir sesi:
- `LimitRange/default-limit` + `ResourceQuota/quota-dev` (dari Modul 5) **masih aktif** — dibiarkan,
  tidak mengganggu (replika HPA min 3 max 6 jauh dari `pods: 20`).
- Sampah drill Modul 5 (`test-noreq`, `quota-test`, `polinux-stress`, bare pod `nginx-demo`) **sudah
  dihapus**. Sampah drill Modul 6 (`load-generator`, `load-generator-2`, `load-generator-3`) **sudah
  dihapus**.
- **metrics-server terpasang** (`kube-system`, patched dengan `--kubelet-insecure-tls` — lihat
  `knowledge/runbooks/kubernetes-cross-cloud-kubeadm-flannel-over-tunnel.md`). `kubectl top nodes`
  berfungsi.
- Aktif dari Modul 6, dibiarkan untuk lanjut/referensi sesi berikutnya: Deployment `nginx-demo`
  (3 replika, `resources.requests` 50m/128Mi + `limits` 100m/256Mi), Service `nginx-demo`
  (ClusterIP 8080→80), HPA `nginx-demo` (min 3 max 6, CPU 70%/memory 75%).
- **Utang teknis kecil:** `deployment/deployment.yaml` di git masih punya field `replicas: 3`
  hardcode — anti-pattern yang dibahas sendiri di sesi ini (konflik nulis field dengan HPA kalau
  file ini di-reapply). Belum diputuskan/dihapus — putuskan di sesi berikutnya (hapus field itu,
  atau biarkan sebagai bahan drill "tarik-tambang" kalau mau didemokan lagi).
- Juga masih ada `podinfo`/`whoami` Deployment (3 replika masing-masing) dari sesi kubeadm
  cross-cloud 2026-08-10 — ini bukan sampah, biarkan.
- Ingress controller **belum diverifikasi ada** di cluster ini — cek dulu sebelum Modul 7
  (`kubectl get pods -A | grep -i ingress`), pasang manual kalau belum ada (pola sama seperti
  metrics-server: cluster ini bare-metal self-managed, tidak ada yang bawaan seperti di RKE2).

**Cluster RKE2/Rancher** (context `local`, terakhir dipakai sampai Modul 4, 2026-08-10 dan
sebelumnya) — state ini **belum diverifikasi ulang**, dicatat sebagai asumsi terakhir:
- **StorageClass `local-path` terpasang permanen** (manifest: `pvc/local-path-provisioner.yaml`),
  sengaja **non-default** — PVC harus eksplisit `storageClassName: local-path`.
- Sisa drill Modul 4 kemungkinan masih jalan di namespace `learning`: Deployment+Service
  `nginx-demo` (readiness `/` sehat + liveness `/salah` rusak → `CrashLoopBackOff`) — bersihkan
  dulu kalau kembali ke cluster ini.
- `pod/pod-with-probe.yaml` di repo sudah di-revert ke kondisi asli (liveness ter-comment) —
  tidak ada aksi git yang tertunda.

## Sesi tambahan (di luar silabus modul) — kubeadm cross-cloud lab, 2026-08-10

Bukan di cluster RKE2 di atas — di cluster kubeadm terpisah (Alibaba + Azure,
1 master + 2 worker, dibahas di `ai-ops/knowledge/runbooks/
kubernetes-cross-cloud-kubeadm-flannel-over-tunnel.md`). Dicatat di sini juga
karena konsepnya generik, bukan spesifik cluster:

- **`kubeadm join` = 3 hal yang harus sinkron:** versi `kubeadm`/`kubelet`/
  `kubectl` (pin exact minor, jangan asal `apt install`), config `containerd`
  (`SystemdCgroup=true` + image `pause` yang dipakai `containerd` harus sama
  persis dengan yang di-pull `kubeadm` — beda versi bikin pod restart random
  tanpa error jelas), dan token+hash dari `kubeadm token create
  --print-join-command` di control-plane (token expired ~24 jam, generate
  baru tiap mau join).
- **Image custom "identik" ≠ behavior identik.** Worker baru dibuat dari image
  yang sama persis dengan master, tapi default login user-nya `root`, bukan
  `ubuntu` seperti master — user `ubuntu` di master ternyata dibuat manual,
  bukan bawaan image. Pelajaran: jangan asumsikan image sama = konfigurasi
  OS-level sama, terutama untuk hal yang biasa dianggap "given" kayak default
  user.
- **kubeconfig = 3 bagian yang saling rujuk by name:** `clusters` (alamat API
  server + CA), `users` (kredensial), `contexts` (pasangan cluster+user+
  namespace, ini yang di-`use-context`). Ganti context cuma ganti pointer,
  tidak ubah isi cluster/user.
- **Sertifikat TLS API server cuma valid untuk alamat yang didaftarkan
  (SAN — Subject Alternative Name) saat cluster dibuat.** Akses dari alamat
  di luar itu (mis. IP publik lewat NAT) ditolak `x509: certificate is valid
  for ..., not ...`. Fix yang benar: tambah SAN baru + regenerate sertifikat
  (`kubeadm init phase certs apiserver`) — BUKAN `insecure-skip-tls-verify`
  di client, itu cuma nutup mata client-nya, server tetap "salah alamat".
  Analoginya: benerin KTP-nya, jangan suruh satpam berhenti cek KTP.
- **Kebiasaan yang sama berlaku lintas cluster:** kubeconfig cluster
  eksperimen tidak pernah digabung ke `~/.kube/config` default (itu cluster
  production client) — selalu file terpisah + `KUBECONFIG` env var, sama
  seperti pola `belajar.config` untuk cluster RKE2 di atas.

## Cara melanjutkan di perangkat lain
1. `git pull` repo ini — instruksi tutor ikut terbawa: `AGENTS.md` (semua AI tool) +
   skill `k8s-belajar` (`.claude/skills/k8s-belajar/`, khusus Claude Code).
2. Pastikan kubeconfig cluster latihan yang mau dipakai tersedia di perangkat itu (file
   terisolasi per cluster); path-nya per-mesin dan tidak disimpan di repo. Sebutkan cluster
   mana yang dipakai saat memulai sesi — AI akan menanyakannya kalau belum disebut.
3. Buka AI tool pilihan di repo ini, minta **"lanjut belajar Kubernetes"**:
   - **Claude Code** → skill `k8s-belajar` terdeteksi otomatis;
   - **tool lain** (Codex, Gemini CLI, Cursor, dll.) → tool membaca `AGENTS.md`.
   Keduanya membaca file ini dan melanjutkan dari checkpoint ⏭️ terakhir.
