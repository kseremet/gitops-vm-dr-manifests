# **GitOps VM DR \- Manifests Repository README**

This repository is the single source of truth for the Disaster Recovery (DR) state of all OpenShift Virtualization VMs.

**WARNING: DO NOT MANUALLY EDIT THE `base/` DIRECTORY\!**

The contents of the `base/` directory, all per-namespace `overlays/`, and the `applicationset.yaml` file are **100% managed by the Ansible DR automation playbooks**. Manual edits will be overwritten during the next reconciliation.  
The **only file** you should ever edit manually is the central control panel Kustomization file, as described in the "Failover Workflow" section.

## **GitOps Model: The "Patch Library"**

This repository uses a Kustomize "Patch Library" model to manage the DR state.

* **`applicationset.yaml`:** This file configures ArgoCD to use a Git Directory generator. It automatically finds every namespace in the `base/` directory and creates a dedicated ArgoCD `Application` for it.
  
* **`base/\<namespace\>/`:** Contains the "power-neutral" manifests (VM, PVC, PV) for all DR-enabled VMs in a given namespace.  

* **`overlays/\<cluster\>/\<namespace\>/`:** This is the overlay that the ArgoCD `Application` points to. It simply includes the `base/` resources and the shared `failover-control` component.  

* **`overlays/\<cluster\>/components/failover-control/`:** This is the **central control panel**. This single Kustomize `Component` is included by *all* namespace applications. It contains a list of patches.  

* **`overlays/\<cluster\>/components/patches/`:** This is the "Patch Library." It contains reusable patches like `power-on.yaml` and `power-off.yaml`.

## **Disaster Recovery (Failover) Workflow**

Failover is a controlled, two-stage process.

### **1\. Prepare Storage (Ansible)**

* An operator must first run the `failover.yml` Ansible playbook.  

* This playbook reads the manifests from the Git repository to identify the target VMs and their associated PVs based on the selector.  It then runs the necessary storage commands (e.g., `rbd mirror image promote --force` for Ceph) for the selected PVs to make them writable.  
  **Future:** *The design allows for adding other storage backends by creating corresponding Ansible task files.*  

* **This step is mandatory.** The VMs will not start if the storage is not promoted.

### **2\. Activate Failover (GitOps)**

This is the only manual change you should make in this repository.

1. Edit the Control Panel: Open the central control panel file:  
   `vms/overlays/\<cluster\_name\>/components/failover-control/kustomization.yaml`  

2. **Activate Patches:** This file contains a list of pre-defined, commented-out patch blocks. To fail over an application group, **uncomment** the corresponding block.
     
   **Before (Standby State):**
    ```yaml
    patches:
      # Default state: ensure everything is off.
      - path: patches/power-off.yaml
        target:
          kind: VirtualMachine
          group: kubevirt.io
    
      # --- FAILOVER GROUPS (Activate by uncommenting) ---
      # - path: patches/power-off.yaml
      #   target:
      #     kind: VirtualMachine
      #     group: kubevirt.io
      #     labelSelector: "app-tier=database"
    ```
  
   **After (Failing over the Databases):**  
    ```yaml
    patches:
      # Default state: ensure everything is off.
      - path: patches/power-off.yaml
        target:
          kind: VirtualMachine
          group: kubevirt.io
    
      # --- FAILOVER GROUPS (Activate by uncommenting) ---
      - path: patches/power-off.yaml
        target:
          kind: VirtualMachine
          group: kubevirt.io
          labelSelector: "app-tier=database"
    ```

3. **Commit & Push:** Commit this one-line change (ideally via a Pull Request). ArgoCD will detect the change, Kustomize will apply the `power-on.yaml` patch to all VMs matching the `app-tier=database` label, and those VMs will start on the DR cluster.

## **Failback Workflow (Conceptual)**

1. **Deactivate Failover (GitOps):** Edit the `failover-control/kustomization.yaml` file and **comment out** the active patch block. Commit this change. ArgoCD will sync and power down the VMs. 

2. **Prepare Storage (Ansible):** Run the `failback.yml` Ansible playbook to demote the DR storage and resync replication from the primary cluster.

## **Directory Structure**
```
└── vms/  
    ├── applicationset.yaml   # (AUTOGEN) Manages all applications  
    ├── base/                 # (AUTOGEN) Power-neutral manifests  
    │   └── <namespace>/  
    │       ├── kustomization.yaml  
    │       ├── vm-*.yaml  
    │       ├── pvc-*.yaml  
    │       └── pv-*.yaml  
    └── overlays/             # Kustomize Overlays  
        └── <cluster_name>/  
            ├── <namespace>/  # (AUTOGEN) ArgoCD App points here  
            │   └── kustomization.yaml  
            └── components/  
                ├── failover-control/  
                │   └── kustomization.yaml # (MANUAL) The DR Control Panel  
                └── patches/  
                    ├── power_on.yaml  
                    └── power_off.yaml
```
