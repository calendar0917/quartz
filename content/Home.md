o> [!quote] 📅 日志
> ```dataview
> LIST FROM "00_日志"
> SORT file.name DESC
> LIMIT 3
> ```

待办

- [ ] 12.10 信安数基 14:00 2h
- [ ] 12.19 计组 14:00 2h
- [ ] 12.20 密码学pre
- [ ] 12.25 C++ 14:00 2h
- [ ] 12.26 密码学 14:00 2h
- [ ] 12.31 数据结构 14:00 2h
- [ ] 1.5 概率论 14:00 2h
- [ ] 1.13 大物 9:30 2h
---

> [!col]
> >> [!todo] 🚧 活跃项目 (Active Processes)
> > 
> > ```dataview
> > TABLE WITHOUT ID file.link AS "Project", status AS "Status", file.mtime AS "Last Mod"
> > FROM "02_进行中"
> > WHERE file.name != this.file.name
> > SORT file.mtime DESC
> > LIMIT 5
> > ```

---

> [!example] 🧠 近期知识录入 (Latest in Kernel)
>
> ```dataview
> TABLE WITHOUT ID file.link AS "Note", file.folder AS "Category", file.ctime AS "Created"
> FROM "01_知识库"
> SORT file.ctime DESC
> LIMIT 5
> ```

---

## 计划中
### 技术

- [ ] CSAPP 前六章
- [ ] OSTEP
- [ ] 自顶向下网络
- [ ] Go
- [ ] Docker 核心原理
### 阅读

- [ ] **黑客：计算机革命的英雄**
- [ ] **若为自由故：自由软件之父理查德·斯托曼传**
- [ ] **监控资本主义时代**
- [ ] **毫无意义的工作**
- [ ] **控制论**