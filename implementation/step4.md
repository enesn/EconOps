# Step 4: Choosing a data format [DL]
I came across Paul Goldsmith-Pinkham’s mini series on Claude Code for applied economists, and discovered Parquet format. What stood out was the format’s ability to embed rich metadata directly within the dataset itself. This would allow me to inspect variable definitions or a codebook inside the dataset without having to look at an external file separately. 

Second thing that impressed me was the size of the compression: If a dataset is 10 GB as CSV, it will often become ~1–3 GB in Parquet! So on an 8 or 16 or even 32 GB RAM machine, this matters a lot. CSV/DTA often does not fit comfortably in memory once loaded, while Parquet often fits easily or can be partially loaded calling only needed columns without reading the entire file. So use Parquet!
