# Challenge 09 — Persistent Storage

[< Previous Challenge](./Challenge-08.md) — **[Home](../README.md)** — [Next Challenge >](./Challenge-10.md)

## Introduction

Pods are disposable, but application data is not. In this challenge you will add durable storage to FabTech so the database keeps its data across pod restarts and shared application content can be accessed by more than one pod.

> **Note:** This challenge is a storage deep-dive. In Challenge 03 you deployed in-cluster PostgreSQL via the **Bitnami Helm chart**, which provisioned its storage through its own PersistentVolumeClaim — the storage details were abstracted away for you. Here you deploy PostgreSQL as an **explicit StatefulSet with your own StorageClass** so you can observe PVC provisioning, Azure Disk CSI attachment, and data survival across pod restarts first-hand. In production, you would use a managed **Azure Database for PostgreSQL Flexible Server** rather than either in-cluster option.

## Description

- Replace the temporary database storage with dynamically provisioned persistent storage backed by Azure Disks.
- Use a CSI-backed storage class that is appropriate for a stateful database workload.
- Ensure the database workload is using a persistent volume claim rather than ephemeral container storage.
- Validate persistence by creating application data, recreating the database pod, and confirming that the data remains available.
- Add a separate shared storage path backed by Azure Files for content that must be mounted read-write by multiple pods at the same time.
- Update the application design so the shared storage can be accessed from more than one workload.
- Compare the access mode requirements for database storage versus shared application files.

## Hints

- Azure Disk is typically the right fit for a single-writer database volume.
- Azure Files is designed for shared access across multiple pods and nodes.
- ReadWriteOnce and ReadWriteMany describe different storage behaviors and should influence your design choices.
- Dynamic provisioning should create the backing storage resource when the claim is created.

## Notes

- NOTE: The database scenario in this challenge should use managed disk-backed persistent storage, not temporary node storage.
- NOTE: Shared read-write storage is a separate requirement from database persistence and usually needs a different storage type.
- NOTE: Test persistence with a pod recreation event, not only with an application restart inside the same pod.

## Optional Advanced

- Protect the persistent volumes with Azure Backup for AKS and review the restore workflow.
- Compare standard and premium storage classes for cost and performance trade-offs.
- Discuss how backup and restore expectations differ for databases versus shared file content.

## Success Criteria

1. The database workload uses a dynamically provisioned persistent volume claim backed by Azure Disk.
2. Database data survives deletion and recreation of the database pod.
3. A shared Azure Files-backed claim is available to multiple pods with read-write access.
4. You can explain to your coach when to choose Azure Disk, Azure Files, ReadWriteOnce, and ReadWriteMany.

## Learning Resources

- [Storage options for applications in AKS](https://learn.microsoft.com/azure/aks/concepts-storage)
- [Use Azure Disk CSI driver in AKS](https://learn.microsoft.com/azure/aks/azure-disk-csi)
- [Use Azure Files CSI driver in AKS](https://learn.microsoft.com/azure/aks/azure-files-csi)
- [Azure Kubernetes Service backup overview](https://learn.microsoft.com/azure/backup/azure-kubernetes-service-backup-overview)
