# Kubernetes Learning — Progress & Recap

Catatan belajar Kubernetes terstruktur, dipakai lintas perangkat (pull repo → buka Claude CLI
→ lanjut dari sini). Panduan modul ada di skill `k8s-belajar`.

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

### ⏭️ Berikutnya — Modul 2: Config & Storage
ConfigMap (config non-rahasia), Secret (data sensitif), PVC (penyimpanan persisten).

## Cara melanjutkan di perangkat lain
1. `git pull` repo ini.
2. Pastikan kubeconfig cluster `learning` tersedia di perangkat itu (context `local`).
3. Buka Claude CLI, minta "lanjut belajar Kubernetes" — skill `k8s-belajar` + file ini jadi acuan.
