---
author: alkohli
ms.service: databox
ms.subservice: pod   
ms.topic: include
ms.date: 08/02/2021
ms.author: alkohli
---

Here is a list of the supported storage accounts and storage types for a Data Box device. For a complete list of all capabilities for all types of storage accounts, see [Types of storage accounts](../articles/storage/common/storage-account-overview.md#types-of-storage-accounts).

For import orders, following table shows the supported storage accounts.

| **Storage account / Supported storage types** | **Block blob** |**Page blob*** |**Azure files** |**Notes**|
| --- | --- | -- | -- | -- |
| Classic Standard | Y | Y | Y |
| General-purpose v1 Standard  | Y | Y | Y | Both hot and cool are supported.|
| General-purpose v1 Premium  |  | Y| | |
| General-purpose v2 Standard  | Y | Y | Y | Both hot and cool are supported.|
| General-purpose v2 Premium  |  |Y | | |
| Azure Premium FileStorage |  |  | Y |  |  
| Blob storage Standard |Y | | |Both hot and cool are supported. |

\* *- Data uploaded to page blobs must be 512 bytes aligned such as VHDs.*

For export orders, following table shows the supported storage accounts.

| **Storage account / Supported storage types** | **Block blob** |**Page blob*** |**Azure files** |**Supported access tiers**|
| --- | --- | -- | -- | -- |
| Classic Standard | Y | Y | Y | |
| General-purpose v1 Standard  | Y | Y | Y | Hot, Cool|
| General-purpose v1 Premium  |  | Y| | |
| General-purpose v2 Standard  | Y | Y | Y | Hot, Cool|
| General-purpose v2 Premium  |  |Y | | |
| Azure Premium FileStorage |  |  | Y |  |
| Blob storage Standard |Y | | |Hot, Cool |
| Block Blob storage Premium |Y | | |Hot, Cool |
| Page Blob storage Premium | |Y | | |

> [!IMPORTANT]
> - For General-purpose accounts, Data Box does not support Queue, Table, and Disk storage types for import orders. For export orders, Data Box does not support Queue, Table, Disk, and Azure Data Lake Gen 2 storage types for General-purpose accounts.
> - Data Box does not support append blobs for Blob Storage and Block Blob Storage accounts.
> - Network File System (NFS) 3.0 protocol support in Azure Blob storage is not supported with Data Box.
> - Data uploaded to page blobs must be 512 bytes aligned such as VHDs.
> - A maximum of 80 TB can be exported.
> - File history and blob snapshots are not exported.
> - Archive blobs are not supported for export. Rehydrate the blobs in archive tier before you export. For more information, see [Rehydrate an archived blob to an online tier](../articles/storage/blobs/archive-rehydrate-overview.md).