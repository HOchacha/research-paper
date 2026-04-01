# nw-research

A repository for collecting and organizing recent papers in networking and systems security, systematically building up research trends and background knowledge.

## What is this repo

- **Purpose**: Read papers in areas of interest — network monitoring, protocols, security, container security — and organize key content into structured markdown notes.
- **Scope**: Archive paper PDFs + write detailed summaries (MD) for selected papers + synthesize research trends and background knowledge across topics.
- **Use**: Explore research ideas, survey related work, and track connections between technical keywords.

## How papers are collected

Papers are collected by filtering with **ACM CCS (Computing Classification System)** categories via Advanced Search on the [ACM Digital Library](https://dl.acm.org/).

### Search links by category (5-year window)

| Topic | CCS Categories | Link |
|-------|---------------|------|
| **Network Protocols** | Transport protocols, Network protocols, Application layer protocols, Cross-layer protocols | [Search](https://dl.acm.org/action/doSearch?fillQuickSearch=false&target=advanced&expand=dl&field1=AllField&CCSOr=92&CCSOr=292&CCSOr=289&CCSOr=294&EpubDate=%5B20210105+TO+20260105%5D) |
| **Network Monitoring** | Network monitoring | [Search](https://dl.acm.org/action/doSearch?fillQuickSearch=false&target=advanced&expand=dl&field1=AllField&CCSOr=324&EpubDate=%5B20210105+TO+20260105%5D) |
| **Network Structure & Security** | Network structure, Network security | [Search](https://dl.acm.org/action/doSearch?fillQuickSearch=false&target=advanced&expand=dl&field1=AllField&CCSOr=310&CCSOr=312&EpubDate=%5B20210105+TO+20260105%5D) |
| **Attacks & Protocol Security** | Denial-of-service attacks, Web protocol security, Security protocols | [Search](https://dl.acm.org/action/doSearch?fillQuickSearch=false&target=advanced&expand=dl&field1=AllField&CCSOr=1004&CCSOr=1007&CCSOr=1005&EpubDate=%5B20210105+TO+20260105%5D) |
| **Trusted Computing & Virtualization Security** | Trusted computing, Virtualization and security | [Search](https://dl.acm.org/action/doSearch?fillQuickSearch=false&target=advanced&expand=dl&field1=AllField&CCSOr=977&CCSOr=978&EpubDate=%5B20210105+TO+20260105%5D) |
| **Distributed Systems Security** | Distributed systems security | [Search](https://dl.acm.org/action/doSearch?fillQuickSearch=false&target=advanced&expand=dl&field1=AllField&CCSOr=259&EpubDate=%5B20210105+TO+20260105%5D) |

> Adjust the date range in the `EpubDate` parameter to change the search window.

## Repo structure

```
papers/                                    # Paper PDFs + summary notes (MD)
├── network monitoring/                    # Network monitoring, fault diagnosis, traffic measurement
├── network security & network structure/  # Network security, topology
├── container security/                    # Container security, OS-level virtualization
└── network protocol/                      # Network protocols

Summary/                                   # Cross-paper analysis and synthesis
├── 관심사.md                               # Per-paper highlights, follow-up questions, ideas
└── 네트워크 연구 배경지식 정리.md             # Background knowledge and research trends
```

- Each subdirectory under `papers/` contains paper PDFs alongside detailed summary notes named `[Venue] Title.md` for selected papers.
- `Summary/hocha-interests.md` accumulates interesting techniques, ideas, and follow-up questions discovered while reading.
- `Summary/network-research-background.md` synthesizes background knowledge and research trends across all reviewed papers.
