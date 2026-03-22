# booking-service

Kubernetes manifests for the Phoenix booking API (`base/` + `overlays/dev` and `overlays/prod`). Includes a **MongoDB StatefulSet** (`booking-service-mongodb`) with its own PVC. Wired from `environments/dev` and `environments/prod`.
