<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=6,11,20&height=200&section=header&text=Nitin%20Jain&fontSize=60&fontColor=fff&animation=twinkling&fontAlignY=35&desc=Platform%20Engineer%20%7C%20Cloud%20Architect%20%7C%20Open-Source%20Builder&descAlignY=55&descSize=18" width="100%" />
</p>

<p align="center">
  <a href="https://git.io/typing-svg">
    <img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=22&pause=1000&color=00B4D8&center=true&vCenter=true&width=750&height=45&lines=Platform+Engineer.+15+years.+Still+shipping.;Kubernetes+%C2%B7+EKS+%C2%B7+OpenShift+%C2%B7+Linkerd+%C2%B7+KEDA;GitOps+with+FluxCD+and+ArgoCD+at+scale;Policy+as+Code+%E2%80%94+OPA+%C2%B7+Kyverno+%C2%B7+Gatekeeper;AI%2FLLM+workflows+%C2%B7+Dynatrace+%C2%B7+Anomaly+Detection;CDN+%C2%B7+WAF+%C2%B7+DNS+%C2%B7+Edge+Functions+at+org+scale;I+build+platforms+teams+trust." alt="Typing SVG" />
  </a>
</p>

<p align="center">
  <img src="https://komarev.com/ghpvc/?username=nitinjain999&label=Profile+Views&color=00B4D8&style=flat" alt="profile views" />
  &nbsp;
  <a href="https://github.com/nitinjain999?tab=followers">
    <img src="https://img.shields.io/github/followers/nitinjain999?label=Followers&style=social" />
  </a>
</p>

<img src="https://capsule-render.vercel.app/api?type=soft&color=gradient&customColorList=6,11,20&height=3&section=header" width="100%" />

---

<p align="center">
  I'm Nitin. I've been doing this for 15 years and I still find it genuinely interesting.<br/>
  My day job is making sure the platform stays out of the way — fast deploys, safe infra, no surprises at 2am.<br/>
  I own CDN, DNS, WAF, and edge functions at org scale. I write Terraform like I mean it.<br/>
  I've broken enough things in prod to know how to build them so they don't break again.
</p>

---

## ⚡ What I Actually Work On

| | |
|--------|------------|
| ☁️ **Edge & CDN** | I manage global CDN, WAF, and DNS for an entire org. When something goes wrong at the edge, it's my phone. |
| ⚙️ **Platform Engineering** | Kubernetes clusters on EKS and OpenShift, Linkerd for mTLS, KEDA for event-driven scaling — the whole stack teams build on top of. |
| 🛡️ **Policy as Code** | I don't just write policies — I make sure they're enforced at admission time, not discovered in a post-incident review. OPA, Kyverno, Gatekeeper. |
| 🔒 **Cloud & IAM** | 40+ account AWS org, built from scratch. OIDC everywhere, no long-lived keys, Terraform modules that actually get reused. |
| 🤖 **AI & Observability** | I build AI-assisted workflows and Claude Code skills for platform work. Dynatrace for observability, anomaly detection before users notice. |

---

## 🚀 Something I Built

### [platform-skills](https://github.com/nitinjain999/platform-skills)

I got tired of Claude giving generic platform advice, so I built a proper skill for it. It knows Kubernetes, Terraform, GitOps, KEDA, Linkerd, OPA, Kyverno, AWS — the actual stuff you deal with in production. Every pattern in there came from something real that happened.

[![Stars](https://img.shields.io/github/stars/nitinjain999/platform-skills?style=social)](https://github.com/nitinjain999/platform-skills)
[![Release](https://img.shields.io/github/v/release/nitinjain999/platform-skills)](https://github.com/nitinjain999/platform-skills/releases)
[![License](https://img.shields.io/github/license/nitinjain999/platform-skills)](https://github.com/nitinjain999/platform-skills/blob/main/LICENSE)

---

## 🧰 Tech Stack

<p align="left">
  <a href="https://skillicons.dev">
    <img src="https://skillicons.dev/icons?i=kubernetes,aws,azure,terraform,prometheus,grafana,githubactions,python,go&perline=9" />
  </a>
</p>

<p align="left">
  <img src="https://img.shields.io/badge/Linkerd-2563EB?style=flat&logo=linkerd&logoColor=white" />
  <img src="https://img.shields.io/badge/KEDA-326CE5?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/FluxCD-5468FF?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white" />
  <img src="https://img.shields.io/badge/OPA-7D3C98?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/Kyverno-326CE5?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/Gatekeeper-FF6B35?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/Dynatrace-1496FF?style=flat&logo=dynatrace&logoColor=white" />
  <img src="https://img.shields.io/badge/Splunk-000000?style=flat&logo=splunk&logoColor=white" />
  <img src="https://img.shields.io/badge/OpenShift-EE0000?style=flat&logo=redhatopenshift&logoColor=white" />
  <img src="https://img.shields.io/badge/EKS-FF9900?style=flat&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Claude_Code-D97757?style=flat&logoColor=white" />
  <img src="https://img.shields.io/badge/LLM_Workflows-00B4D8?style=flat&logoColor=white" />
</p>

---

## 🏗️ Infrastructure as Code

> If you're clicking around in the AWS console to make changes, we need to talk.

Infrastructure is just code. It gets reviewed, tested, promoted through environments, and tracked in Git. I've seen what happens when it isn't — I've cleaned up enough of those messes.

| | |
|----------|---------------|
| **Terraform** | Proper modules, shared across 40+ accounts. State in S3 with locking. CI pipelines use OIDC — nobody has a secret key sitting in their `.env`. |
| **GitOps** | FluxCD watches Git, not a human. Kustomize overlays keep environments honest. HelmRelease objects so Helm upgrades go through the same review as everything else. |
| **Policy as Code** | Kyverno catches bad manifests at admission — before they land, not after. OPA/Gatekeeper for the tricky cross-namespace stuff. |
| **Helm** | Internal platform charts with schema-validated values. If `helm unittest` doesn't pass, it doesn't ship. |
| **Secrets** | External Secrets Operator pulling from AWS Secrets Manager. Nothing sensitive touches Git. If I find plaintext in a repo, it's an incident. |
| **State hygiene** | `terraform plan` is mandatory in CI, blast radius is part of every review, and service boundaries have their own state files. One mistake shouldn't take down everything. |

<p align="left">
  <img src="https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white" />
  <img src="https://img.shields.io/badge/FluxCD-5468FF?style=for-the-badge&logoColor=white" />
  <img src="https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white" />
  <img src="https://img.shields.io/badge/Kustomize-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" />
  <img src="https://img.shields.io/badge/OPA-7D3C98?style=for-the-badge&logoColor=white" />
  <img src="https://img.shields.io/badge/Kyverno-326CE5?style=for-the-badge&logoColor=white" />
  <img src="https://img.shields.io/badge/External_Secrets-FF6B35?style=for-the-badge&logoColor=white" />
  <img src="https://img.shields.io/badge/AWS_CDK-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
</p>

---

## 📊 GitHub Stats

<p align="center">
  <img src="https://github-readme-stats.vercel.app/api?username=nitinjain999&show_icons=true&theme=tokyonight&hide_border=true&count_private=true&include_all_commits=true&rank_icon=github" height="180" />
  &nbsp;
  <img src="https://streak-stats.demolab.com?user=nitinjain999&theme=tokyonight&hide_border=true" height="180" />
</p>

<p align="center">
  <img src="https://github-readme-stats.vercel.app/api/top-langs/?username=nitinjain999&layout=donut&theme=tokyonight&hide_border=true&langs_count=8&hide=html,css" height="220" />
  &nbsp;
  <img src="https://github-profile-summary-cards.vercel.app/api/cards/productive-time?username=nitinjain999&theme=tokyonight&utcOffset=1" height="220" />
</p>

<p align="center">
  <img src="https://github-profile-trophy.vercel.app/?username=nitinjain999&theme=tokyonight&no-frame=true&row=1&column=7&margin-w=10" />
</p>

---

## 🐍 Contribution Activity

<p align="center">
  <img src="https://github-readme-activity-graph.vercel.app/graph?username=nitinjain999&theme=tokyo-night&hide_border=true&area=true&area_color=00B4D8&color=00B4D8&line=00B4D8&point=FFFFFF" width="100%" />
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/nitinjain999/nitinjain999/output/github-contribution-grid-snake-dark.svg" />
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/nitinjain999/nitinjain999/output/github-contribution-grid-snake.svg" />
    <img alt="github-snake" src="https://raw.githubusercontent.com/nitinjain999/nitinjain999/output/github-contribution-grid-snake-dark.svg" />
  </picture>
</p>

---

## 🤝 Let's Connect

If you're building a platform, untangling a gnarly Terraform state, or just want to talk through a GitOps architecture — reach out. I like these conversations.

<p align="left">
  <a href="https://github.com/nitinjain999">
    <img src="https://img.shields.io/github/followers/nitinjain999?label=Follow+on+GitHub&style=social" />
  </a>
  &nbsp;
  <a href="mailto:nitin.solna@gmail.com">
    <img src="https://img.shields.io/badge/Email-nitin.solna%40gmail.com-D14836?style=flat&logo=gmail&logoColor=white" />
  </a>
</p>

<img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=6,11,20&height=100&section=footer" width="100%" />
