# Step 3: Choosing a storage platform [DL] 
While my current data size is still quite manageable, reaching 5 TB is realistic over time if I continuously append new data waves, retain historical snapshots, or duplicate merged datasets.

For my scale, a simple file-storage workflow using Dropbox can still remain sufficient for quite a long time if datasets are partitioned properly. Yet, I still wanted to compare the alternatives with realistic use cases:

| Profile | Storage | Egress | AWS S3 | Cloudflare R2 | B2 | Dropbox |
| --- | --- | --- | --- | --- | --- | --- |
| Microdata  | 50 GB | 0.5 TB | $37.15 | $0.75 | $3.85 | — |
| Retailer scanner (subset) | 500 GB | 5 TB | $452.50 | $7.50 | $38.48 | — |
| Firm-level: Compustat | 10 GB | 0.1 TB | $0.23 | $0.15 | $0.77 | — |
| Firm-level: Orbis | 250 GB | 2.5 TB | $221.75 | $3.75 | $19.24 | — |
| **TOTAL MONTHLY** | **810 GB** | **8.1 TB** | **$711.63** | **$12.15** | **$62.33** | **$11.99** |

From a cost standpoint, Dropbox still appears to be a reasonable choice. It is also widely used among researchers and economists, and it supports simple OAuth-based authentication (see below) that keeps data fully cloud-hosted while allowing access through user-authorized short- or long-lived tokens. For our current needs and scale, moving to a dedicated object storage system would likely be overkilling.

Since Dropbox already stores our family photos, videos, and documents, I expect to approach its 5 TB plan limit in about five years. Once we cross that threshold, it will make sense to reassess storage options. For example, it is possible to assemble a ~20 TB storage setup for around $1,500. But this would introduce important tradeoffs: we would lose the reliability, scalability, and global accessibility of managed services like R2, while also taking on responsibilities such as hardware maintenance, drive failures, and electricity costs. 

So TL;DR: Stick with Dropbox up to 5TB, then transition to R2 once you exceed that limit.
