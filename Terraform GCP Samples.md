# Terraform GCP Samples
### 1. VPC
```
resource "google_compute_network" "vpc" {
  name                    = "demo-vpc"
  auto_create_subnetworks = false
}
```
### 2. Subnets
```
resource "google_compute_subnetwork" "subnet" {
  name                  = "demo-subnet"
  ip_cidr_range         = "10.0.1.0/24"
  region                = "us-central1"
  network               = google_compute_network.vpc.id
  private_ip_google_access = true
}
```
### 3. GCE VM (Compute Engine VM)
```
resource "google_compute_instance" "vm" {
  name         = "demo-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "projects/debian-cloud/global/images/family/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id

    access_config {}  # For public IP
  }
}
```
### 4. Cloud Storage Bucket
```
resource "google_storage_bucket" "bucket" {
  name     = "demo-storage-bucket-1234"
  location = "US"
  force_destroy = true
}
```
### 5. GKE Cluster (Google Kubernetes Engine)
```
resource "google_container_cluster" "gke" {
  name     = "demo-gke"
  location = "us-central1"

  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-nodes"
  location   = "us-central1"
  cluster    = google_container_cluster.gke.name

  node_config {
    machine_type = "e2-medium"
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }

  initial_node_count = 1
}
```
### 6. IAM Service Account
```
resource "google_service_account" "sa" {
  account_id   = "demo-service-account"
  display_name = "Demo Service Account"
}
```
