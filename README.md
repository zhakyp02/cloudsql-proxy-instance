# cloudsql-proxy-instance
root module main.tf
```hcl
module "gce-advanced-container" {
  source = "terraform-google-modules/container-vm/google"

  container = {
    image = "gcr.io/cloudsql-docker/gce-proxy:latest"
    command = [
      "/cloud_sql_proxy"
    ]
    args = ["-instances=${module.cloudsql_postgres_sync_test.connection_name}=tcp:0.0.0.0:5432"]
    securityContext = {
      privileged = true
    }
    tty = true
    env = [
      {
        name  = "LOGFILE_MAX_SIZE"
        value = "2000"
      },
      {
        name  = "LOGFILE_MAX_COUNT"
        value = "5"
      }
    ]
  }

  restart_policy = "OnFailure"
}

module "cloud_sql_proxy" {
  source     = "../../child/proxy-child"
  project_id = var.project_id
  region     = var.region
  zone       = var.zones[0]
  proxy_subnet      = var.subnet_link
  user_labels       = var.user_labels
  sql_instance_name = module.cloudsql_postgres_sync_test.instance_name
  connection_name   = module.cloudsql_postgres_sync_test.connection_name
  proxy_gce         = module.gce-advanced-container.metadata_value
}
```
child module cloud sql proxy main.tf 

```hcl

resource "google_compute_instance" "sql_proxy_gce_instance" {
  project      = local.project_id
  name         = "sql-proxy-${local.instance_name}"
  machine_type = var.machine_type
  zone         = var.zone

  allow_stopping_for_update = false
  labels                    = var.user_labels

  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }

  network_interface {
    subnetwork = var.proxy_subnet
    # network = "default"
  }

  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  metadata = {
    "google-logging-enabled"  = "true"
    gce-container-declaration = var.proxy_gce

  }

  metadata_startup_script = "echo '{\"live-restore\": true, \"log-opts\":{\"max-size\": \"1kb\", \"max-file\": \"5\" }, \"storage-driver\": \"overlay2\", \"mtu\": 1460}' | sudo jq . | sudo tee /etc/docker/daemon.json >/dev/null; sudo systemctl restart docker; docker run -d gcr.io/stackdriver-agents/stackdriver-logging-agent:1.9.8"


  service_account {
    email = google_service_account.proxy_sa_service_account.email
    scopes = [
      "https://www.googleapis.com/auth/devstorage.read_only",
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring.write",
      "https://www.googleapis.com/auth/service.management.readonly",
      "https://www.googleapis.com/auth/servicecontrol",
      "https://www.googleapis.com/auth/sqlservice.admin",
      "https://www.googleapis.com/auth/trace.append"
    ]
  }

  depends_on = [
    google_service_account.proxy_sa_service_account,
  ]

 
}
```
child module cloud sql proxy variable.tf
```hcl
variable "proxy_gce" {
  type = string
}
```

master instance child module outputs.tf
```hcl
output "connection_name" {
  value = google_sql_database_instance.postgres_db_instance.connection_name
}
```
