### Updated `main.tf` — CentOS 9 + Ports only (no DB installs)

```hcl
provider "google" {
  project = "ams-kap"
  region  = "us-central1"
  # No credentials — uses gcloud ADC
}

# ── VPC ────────────────────────────────────────────────────────
resource "google_compute_network" "ams_kap_vpc" {
  name                    = "ams-kap-vpc"
  auto_create_subnetworks = false
}

# ── SUBNET 1 — us-central1 ─────────────────────────────────────
resource "google_compute_subnetwork" "ams_subnet_1" {
  name          = "ams-subnet-1"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-central1"
  network       = google_compute_network.ams_kap_vpc.id
}

# ── SUBNET 2 — us-east1 ────────────────────────────────────────
resource "google_compute_subnetwork" "ams_subnet_2" {
  name          = "ams-subnet-2"
  ip_cidr_range = "10.0.2.0/24"
  region        = "us-east1"
  network       = google_compute_network.ams_kap_vpc.id
}

# ── SUBNET 3 — europe-west1 ────────────────────────────────────
resource "google_compute_subnetwork" "ams_subnet_3" {
  name          = "ams-subnet-3"
  ip_cidr_range = "10.0.3.0/24"
  region        = "europe-west1"
  network       = google_compute_network.ams_kap_vpc.id
}

# ── FIREWALL — SSH ─────────────────────────────────────────────
resource "google_compute_firewall" "ams_allow_ssh" {
  name    = "ams-allow-ssh"
  network = google_compute_network.ams_kap_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]   # Restrict to your IP in production
  target_tags   = ["ams-server"]
}

# ── FIREWALL — Web ports ────────────────────────────────────────
resource "google_compute_firewall" "ams_allow_web" {
  name    = "ams-allow-web"
  network = google_compute_network.ams_kap_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["ams-server"]
}

# ── FIREWALL — Database ports (internal VPC only) ───────────────
resource "google_compute_firewall" "ams_allow_db" {
  name    = "ams-allow-db-ports"
  network = google_compute_network.ams_kap_vpc.name

  allow {
    protocol = "tcp"
    ports = [
      "5432",  # PostgreSQL
      "3306",  # MySQL / MariaDB
      "6379",  # Redis
      "27017", # MongoDB
      "5672",  # RabbitMQ
      "9200",  # Elasticsearch
      "1521",  # Oracle DB
      "1433",  # MS SQL Server
    ]
  }

  # DB ports open only within VPC — NOT public internet
  source_ranges = ["10.0.0.0/8"]
  target_tags   = ["ams-server"]
}

# ── VM 1 — us-central1 (CentOS 9) ──────────────────────────────
resource "google_compute_instance" "ams_vm_1" {
  name         = "ams-vm-1"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
  tags         = ["ams-server"]

  boot_disk {
    initialize_params {
      image = "centos-cloud/centos-stream-9"
      size  = 20   # GB
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.ams_subnet_1.id
    access_config {}   # Public IP
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    dnf update -y
    # Firewalld — open ports only, no DB install
    systemctl enable firewalld
    systemctl start firewalld
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --permanent --add-port=5432/tcp
    firewall-cmd --permanent --add-port=3306/tcp
    firewall-cmd --permanent --add-port=6379/tcp
    firewall-cmd --permanent --add-port=27017/tcp
    firewall-cmd --reload
  EOF
}

# ── VM 2 — us-east1 (CentOS 9) ─────────────────────────────────
resource "google_compute_instance" "ams_vm_2" {
  name         = "ams-vm-2"
  machine_type = "e2-medium"
  zone         = "us-east1-b"
  tags         = ["ams-server"]

  boot_disk {
    initialize_params {
      image = "centos-cloud/centos-stream-9"
      size  = 20
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.ams_subnet_2.id
    access_config {}
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    dnf update -y
    systemctl enable firewalld
    systemctl start firewalld
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --permanent --add-port=5432/tcp
    firewall-cmd --permanent --add-port=3306/tcp
    firewall-cmd --permanent --add-port=6379/tcp
    firewall-cmd --permanent --add-port=27017/tcp
    firewall-cmd --reload
  EOF
}

# ── VM 3 — europe-west1 (CentOS 9) ─────────────────────────────
resource "google_compute_instance" "ams_vm_3" {
  name         = "ams-vm-3"
  machine_type = "e2-medium"
  zone         = "europe-west1-b"
  tags         = ["ams-server"]

  boot_disk {
    initialize_params {
      image = "centos-cloud/centos-stream-9"
      size  = 20
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.ams_subnet_3.id
    access_config {}
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    dnf update -y
    systemctl enable firewalld
    systemctl start firewalld
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --permanent --add-port=5432/tcp
    firewall-cmd --permanent --add-port=3306/tcp
    firewall-cmd --permanent --add-port=6379/tcp
    firewall-cmd --permanent --add-port=27017/tcp
    firewall-cmd --reload
  EOF
}
```

---

## `outputs.tf` — same as before

```hcl
output "vpc_name" {
  value = google_compute_network.ams_kap_vpc.name
}

output "vm_1_public_ip" {
  value = google_compute_instance.ams_vm_1.network_interface[0].access_config[0].nat_ip
}

output "vm_2_public_ip" {
  value = google_compute_instance.ams_vm_2.network_interface[0].access_config[0].nat_ip
}

output "vm_3_public_ip" {
  value = google_compute_instance.ams_vm_3.network_interface[0].access_config[0].nat_ip
}
```

---

## What changed from previous version

| Item | Before | Now |
|---|---|---|
| OS | `debian-cloud/debian-11` | `centos-cloud/centos-stream-9` |
| Package manager | `apt-get` | `dnf` |
| DB software | Installed PostgreSQL / MySQL | **Not installed** |
| Port management | None | `firewalld` enabled + ports opened |
| DB ports | Only in GCP firewall | GCP firewall + OS-level firewalld |

---

### Deploy

```powershell
cd E:\my-learning\my-terraform\amskap-proj

terraform init
terraform plan
terraform apply
```

After deploy, SSH in and verify ports are open:

```bash
gcloud compute ssh ams-vm-1 --zone=us-central1-a --project=ams-kap

# Check open ports on the VM
firewall-cmd --list-ports
```
