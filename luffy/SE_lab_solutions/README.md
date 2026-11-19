# SE Lab — Question Papers & Solutions

**Exam:** Lab Internal II — Software Engineering (23CC501PC / 23CC505PC), A.Y. 2026-27.
This folder contains all **17 original question papers (.docx)** copied from `SE lab2.zip` and a **self-contained markdown solution file for every paper**.

> The `.docx` files are the **original, unmodified question papers** from the zip. Open the matching `-SOLUTIONS.md` file while practising. Each solution file repeats the full question wording, gives a plain-English explanation, exam-ready steps, exact runnable commands (PowerShell / Groovy / YAML / Dockerfile / docker-compose), and short answers to every scenario-based sub-question. Ports used: Jenkins 8080, Tomcat 8085, Nagios 8888, WordPress 8000.

## Top-level papers (`SE lab2/SET-1.docx` … `SET-9.docx`)

| Paper | Solution file | Questions answered |
|---|---|---|
| `SET-1.docx` | `SET-1-SOLUTIONS.md` | 5 (Docker multi-container, Jenkins Freestyle, Kubernetes, Nagios, AWS EC2) — Digital Library |
| `SET-2.docx` | `SET-2-SOLUTIONS.md` | 5 (Docker Tomcat+MySQL, Jenkins Freestyle, Nagios, Kubernetes scaling, AWS EC2) — Metro Reservation |
| `SET-3.docx` | `SET-3-SOLUTIONS.md` | 5 (Docker Tomcat+MySQL, Jenkins Freestyle, Nagios, Kubernetes scaling, AWS EC2) — AI-OLMS |
| `SET-4.docx` | `SET-4-SOLUTIONS.md` | 4 (Jenkins Scripted Pipeline, Email Notification, Minikube+Nagios, AWS EC2) — Hospital Management |
| `SET-5.docx` | `SET-5-SOLUTIONS.md` | 4 (Docker Compose Flask+Nginx, Jenkins Scripted Pipeline, Kubernetes, AWS EC2) — Food Management |
| `SET-6.docx` | `SET-6-SOLUTIONS.md` | 5 (Docker Compose WordPress+MySQL, Jenkins Scripted Pipeline, Kubernetes, Nagios, AWS EC2) — CEMS |
| `SET-7.docx` | `SET-7-SOLUTIONS.md` | 4 (Docker Compose + MySQL, Jenkins Scripted CI/CD OLTRS, Minikube, AWS EC2) |
| `SET-8.docx` | `SET-8-SOLUTIONS.md` | 4 (Docker Compose + Nagios, Jenkins Pipeline View OLTRS, Minikube, AWS EC2) |
| `SET-9.docx` | `SET-9-SOLUTIONS.md` | 4 (same paper as SET-7 — standalone copy of the answers) |

## 17-11-26 batch (`SE lab2/17-11-26/SET-1.docx` … `SET-8.docx`)

| Paper | Solution file | Questions answered |
|---|---|---|
| `17-11-26\SET-1.docx` | `17-11-26\SET-1-SOLUTIONS.md` | 5 (Docker, Jenkins Freestyle, Kubernetes, Nagios, AWS EC2 + CloudWatch) — E-Commerce |
| `17-11-26\SET-2.docx` | `17-11-26\SET-2-SOLUTIONS.md` | 5 (same flow as SET-1) — Hospital Management |
| `17-11-26\SET-3.docx` | `17-11-26\SET-3-SOLUTIONS.md` | 5 (same flow as SET-1) — E-Learning |
| `17-11-26\SET-4.docx` | `17-11-26\SET-4-SOLUTIONS.md` | 4 (same paper as top-level SET-7 — standalone copy) |
| `17-11-26\SET-5.docx` | `17-11-26\SET-5-SOLUTIONS.md` | 4 (same paper as top-level SET-7 — standalone copy) |
| `17-11-26\SET-6.docx` | `17-11-26\SET-6-SOLUTIONS.md` | 4 (same paper as top-level SET-8 — standalone copy) |
| `17-11-26\SET-7.docx` | `17-11-26\SET-7-SOLUTIONS.md` | 5 (same paper as top-level SET-2 — standalone copy) |
| `17-11-26\SET-8.docx` | `17-11-26\SET-8-SOLUTIONS.md` | 4 (same paper as top-level SET-5 — standalone copy) |

## Totals

- **17 question papers** (.docx) + **17 solution files** + this README = **35 files**.
- **76 main questions answered**, plus all scenario-based sub-questions (5 per 20-mark question; 5 sub-parts per 25-mark question).

## Machine-specific facts used in the answers

- Docker images already pulled: `mysql:8.0`, `wordpress:latest`, `jasonrivers/nagios:latest`.
- Jenkins at `http://localhost:8080` (no login); Maven tool name **`MAVEN-HOME`**, JDK tool **`JDK21`**.
- Minikube start: `minikube start --driver=docker` (Kubernetes v1.37.0); `kubectl` available.
- Nagios: `docker run --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest` → `http://localhost:8888`, `nagiosadmin` / `nagios`.
- Tomcat on port 8085 with manager user `admin` / `1234`. WordPress on **8000** (never 8080 while Jenkins runs).
- Repos: `library-management` (main, JAR), `OLTRS` (master, WAR, empty Dockerfile — write it), `metro-reservation`, `AI-OLMS`, `HospitalManagementSystem`, `FoodSystem`, `CEMS`, and the `17-11-26` repos.
- Full verified command reference: `..\SE_LAB_SETUP_AND_VIVA.md`.
