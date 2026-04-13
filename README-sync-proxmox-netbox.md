# 🔄 Sinkronisasi Proxmox ke NetBox (Ansible Playbook)

Repositori ini berisi Ansible Playbook yang digunakan untuk melakukan sinkronisasi data resource **VM** dan **LXC** dari **Proxmox** ke **NetBox** secara otomatis.

Playbook ini dirancang untuk dijalankan melalui **Ansible Semaphore** maupun secara manual, serta sudah terintegrasi dengan **notifikasi Discord** untuk melaporkan hasil sinkronisasi.

---

# ⚙️ Tujuan Playbook

Playbook ini dibuat untuk:

- menjaga inventaris VM dan LXC di NetBox tetap selaras dengan kondisi aktual di Proxmox
- mendeteksi resource baru yang belum ada di NetBox
- memperbarui data resource yang berubah
- menampilkan kandidat data yang sudah tidak ada di Proxmox sebagai **delete preview**
- mengirim notifikasi hasil sinkronisasi ke Discord
- tetap mengirim notifikasi gagal apabila terjadi error fatal pada proses utama

---

# 🧩 File Utama

Playbook utama pada project ini adalah:

```text
playbooks/sync_proxmox_netbox.yml
