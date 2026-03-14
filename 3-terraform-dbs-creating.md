### Folder Structure for MySQL & PostgreSQL

```
amskap-proj/
├── main.tf
├── outputs.tf
├── variables.tf
└── modules/
    ├── mysql/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── postgres/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Create folders in PowerShell:
```powershell
cd E:\my-learning\my-terraform\amskap-proj
mkdir modules\mysql
mkdir modules\postgres
```

---

## Module 1 — MySQL 8.x

**`modules/mysql/variables.tf`**
```hcl
variable "vm_name" {
  description = "Name of the VM"
  type        = string
}

variable "zone" {
  description = "GCP zone"
  type        = string
}

variable "machine_type" {
  description = "GCP machine type"
  type        = string
  default     = "e2-medium"
}

variable "subnetwork" {
  description = "Subnetwork ID"
  type        = string
}

variable "tags" {
  description = "Network tags"
  type        = list(string)
  default     = ["ams-server"]
}

variable "disk_size" {
  description = "Boot disk size in GB"
  type        = number
  default     = 20
}

variable "mysql_root_password" {
  description = "MySQL root password"
  type        = string
  sensitive   = true
  default     = "AmsKap@MySQL8!"
}
```

**`modules/mysql/main.tf`**
```hcl
resource "google_compute_instance" "mysql_vm" {
  name         = var.vm_name
  machine_type = var.machine_type
  zone         = var.zone
  tags         = var.tags

  boot_disk {
    initialize_params {
      image = "centos-cloud/centos-stream-9"
      size  = var.disk_size
    }
  }

  network_interface {
    subnetwork = var.subnetwork
    access_config {}
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    set -e
    exec > /var/log/startup-mysql.log 2>&1

    echo ">>> Updating system..."
    dnf update -y

    echo ">>> Adding MySQL 8 community repo..."
    dnf install -y https://dev.mysql.com/get/mysql80-community-release-el9-5.noarch.rpm

    echo ">>> Installing MySQL 8..."
    dnf install -y mysql-community-server

    echo ">>> Starting MySQL..."
    systemctl enable mysqld
    systemctl start mysqld

    echo ">>> Getting temp root password..."
    TEMP_PASS=$(grep 'temporary password' /var/log/mysqld.log | awk '{print $NF}')

    echo ">>> Setting new root password..."
    mysql --connect-expired-password -uroot -p"$TEMP_PASS" <<MYSQL_SCRIPT
    ALTER USER 'root'@'localhost' IDENTIFIED BY '${var.mysql_root_password}';
    CREATE DATABASE IF NOT EXISTS ams_db;
    CREATE USER IF NOT EXISTS 'ams_user'@'%' IDENTIFIED BY '${var.mysql_root_password}';
    GRANT ALL PRIVILEGES ON ams_db.* TO 'ams_user'@'%';
    FLUSH PRIVILEGES;
MYSQL_SCRIPT

    echo ">>> Configuring MySQL to accept remote connections..."
    sed -i 's/^bind-address.*/bind-address = 0.0.0.0/' /etc/my.cnf
    systemctl restart mysqld

    echo ">>> Opening firewall ports..."
    systemctl enable firewalld
    systemctl start firewalld
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=3306/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --reload

    echo ">>> MySQL 8 setup complete!"
    mysql -uroot -p"${var.mysql_root_password}" -e "SELECT VERSION();"
  EOF
}
```

**`modules/mysql/outputs.tf`**
```hcl
output "instance_name" {
  value = google_compute_instance.mysql_vm.name
}

output "internal_ip" {
  value = google_compute_instance.mysql_vm.network_interface[0].network_ip
}

output "external_ip" {
  value = google_compute_instance.mysql_vm.network_interface[0].access_config[0].nat_ip
}

output "zone" {
  value = google_compute_instance.mysql_vm.zone
}
```

---

## Module 2 — PostgreSQL 16

**`modules/postgres/variables.tf`**
```hcl
variable "vm_name" {
  description = "Name of the VM"
  type        = string
}

variable "zone" {
  description = "GCP zone"
  type        = string
}

variable "machine_type" {
  description = "GCP machine type"
  type        = string
  default     = "e2-medium"
}

variable "subnetwork" {
  description = "Subnetwork ID"
  type        = string
}

variable "tags" {
  description = "Network tags"
  type        = list(string)
  default     = ["ams-server"]
}

variable "disk_size" {
  description = "Boot disk size in GB"
  type        = number
  default     = 20
}

variable "postgres_password" {
  description = "PostgreSQL password for postgres user"
  type        = string
  sensitive   = true
  default     = "AmsKap@PG16!"
}

variable "postgres_db" {
  description = "Default database name to create"
  type        = string
  default     = "ams_db"
}
```

**`modules/postgres/main.tf`**
```hcl
resource "google_compute_instance" "postgres_vm" {
  name         = var.vm_name
  machine_type = var.machine_type
  zone         = var.zone
  tags         = var.tags

  boot_disk {
    initialize_params {
      image = "centos-cloud/centos-stream-9"
      size  = var.disk_size
    }
  }

  network_interface {
    subnetwork = var.subnetwork
    access_config {}
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    set -e
    exec > /var/log/startup-postgres.log 2>&1

    echo ">>> Updating system..."
    dnf update -y

    echo ">>> Adding PostgreSQL 16 repo..."
    dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

    echo ">>> Disabling built-in postgres module..."
    dnf -qy module disable postgresql

    echo ">>> Installing PostgreSQL 16..."
    dnf install -y postgresql16-server postgresql16-contrib

    echo ">>> Initializing database..."
    /usr/pgsql-16/bin/postgresql-16-setup initdb

    echo ">>> Starting PostgreSQL 16..."
    systemctl enable postgresql-16
    systemctl start postgresql-16

    echo ">>> Setting postgres user password..."
    sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '${var.postgres_password}';"

    echo ">>> Creating database and user..."
    sudo -u postgres psql <<PSQL_SCRIPT
    CREATE DATABASE ${var.postgres_db};
    CREATE USER ams_user WITH ENCRYPTED PASSWORD '${var.postgres_password}';
    GRANT ALL PRIVILEGES ON DATABASE ${var.postgres_db} TO ams_user;
PSQL_SCRIPT

    echo ">>> Allowing remote connections..."
    PG_HBA="/var/lib/pgsql/16/data/pg_hba.conf"
    PG_CONF="/var/lib/pgsql/16/data/postgresql.conf"

    sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" $PG_CONF
    echo "host    all             all             0.0.0.0/0               md5" >> $PG_HBA

    systemctl restart postgresql-16

    echo ">>> Opening firewall ports..."
    systemctl enable firewalld
    systemctl start firewalld
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=5432/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --reload

    echo ">>> PostgreSQL 16 setup complete!"
    sudo -u postgres psql -c "SELECT VERSION();"
  EOF
}
```

**`modules/postgres/outputs.tf`**
```hcl
output "instance_name" {
  value = google_compute_instance.postgres_vm.name
}

output "internal_ip" {
  value = google_compute_instance.postgres_vm.network_interface[0].network_ip
}

output "external_ip" {
  value = google_compute_instance.postgres_vm.network_interface[0].access_config[0].nat_ip
}

output "zone" {
  value = google_compute_instance.postgres_vm.zone
}
```

---

## Root `main.tf` — Call Both Modules

```hcl
provider "google" {
  project = "ams-kap"
  region  = "us-central1"
}

# ── VPC ────────────────────────────────────────────────────────
resource "google_compute_network" "ams_kap_vpc" {
  name                    = "ams-kap-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "ams_subnet_1" {
  name          = "ams-subnet-1"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-central1"
  network       = google_compute_network.ams_kap_vpc.id
}

resource "google_compute_subnetwork" "ams_subnet_2" {
  name          = "ams-subnet-2"
  ip_cidr_range = "10.0.2.0/24"
  region        = "us-east1"
  network       = google_compute_network.ams_kap_vpc.id
}

resource "google_compute_subnetwork" "ams_subnet_3" {
  name          = "ams-subnet-3"
  ip_cidr_range = "10.0.3.0/24"
  region        = "europe-west1"
  network       = google_compute_network.ams_kap_vpc.id
}

# ── FIREWALL ───────────────────────────────────────────────────
resource "google_compute_firewall" "ams_allow_all" {
  name    = "ams-kap-firewall"
  network = google_compute_network.ams_kap_vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22", "80", "443", "3306", "5432"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["ams-server"]
}

# ── MySQL VM — us-central1 ─────────────────────────────────────
module "mysql_server" {
  source     = "./modules/mysql"
  vm_name    = "ams-vm-mysql"
  zone       = "us-central1-a"
  subnetwork = google_compute_subnetwork.ams_subnet_1.id

  mysql_root_password = "AmsKap@MySQL8!"
}

# ── PostgreSQL VM — us-east1 ───────────────────────────────────
module "postgres_server" {
  source     = "./modules/postgres"
  vm_name    = "ams-vm-postgres"
  zone       = "us-east1-b"
  subnetwork = google_compute_subnetwork.ams_subnet_2.id

  postgres_password = "AmsKap@PG16!"
  postgres_db       = "ams_db"
}
```

---

## Root `outputs.tf`

```hcl
output "mysql_internal_ip" {
  value = module.mysql_server.internal_ip
}

output "mysql_external_ip" {
  value = module.mysql_server.external_ip
}

output "postgres_internal_ip" {
  value = module.postgres_server.internal_ip
}

output "postgres_external_ip" {
  value = module.postgres_server.external_ip
}
```

---

## Deploy

```powershell
cd E:\my-learning\my-terraform\amskap-proj

terraform init      # picks up new modules
terraform plan      # preview 2 new VMs
terraform apply     # deploy

# Check outputs
terraform output
```

---

## Verify after deploy

```powershell
# SSH into MySQL VM
gcloud compute ssh ams-vm-mysql --zone=us-central1-a --project=ams-kap

# Check MySQL version
mysql -uroot -p"AmsKap@MySQL8!" -e "SELECT VERSION();"

# SSH into Postgres VM
gcloud compute ssh ams-vm-postgres --zone=us-east1-b --project=ams-kap

# Check Postgres version
sudo -u postgres psql -c "SELECT VERSION();"
```

---

| VM | Zone | DB | Port | Internal IP |
|---|---|---|---|---|
| `ams-vm-mysql` | us-central1-a | MySQL 8.x | 3306 | 10.0.1.x |
| `ams-vm-postgres` | us-east1-b | PostgreSQL 16 | 5432 | 10.0.2.x |
