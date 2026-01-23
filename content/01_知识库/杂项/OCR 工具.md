---
creation date: 2025-12-28 20:44
modification date: 2025-12-28 20:44
---
```bash
docker run --rm \
  -v "$(pwd):/home/docker" \
  --workdir /home/docker \
  jbarlow83/ocrmypdf \
  --sidecar output.txt \
  -l chi_sim+eng \
  "数据结构与算法.pdf" \
  temp.pdf
```