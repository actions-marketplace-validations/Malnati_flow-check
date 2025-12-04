<div align="center">

# 🛡️ Branch Flow Guard

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Branch%20Flow%20Guard-6f42c1?style=for-the-badge&logo=github)](https://github.com/marketplace/actions/branch-flow-guard)
[![Version](https://img.shields.io/github/v/release/Malnati/flow-check?style=for-the-badge&color=purple)](https://github.com/Malnati/flow-check/releases)
[![License](https://img.shields.io/github/license/Malnati/flow-check?style=for-the-badge&color=blue)](LICENSE)

**Imponha regras estritas de Git Flow e valide o conteúdo de PRs automaticamente.**

<p align="center">
  <a href="#-sobre">Sobre</a> •
  <a href="#-regras-do-fluxo">Regras</a> •
  <a href="#-instalação">Instalação</a> •
  <a href="#-inputs--outputs">Inputs & Outputs</a>
</p>

</div>

---

## 🚀 Sobre

O **Branch Flow Guard** é um "porteiro" para seus Pull Requests. Ele analisa a origem e o destino de cada PR para garantir que o ciclo de vida do software seja respeitado (Dev → Staging → Prod).

Além disso, ele detecta inteligentemente se há alterações reais de código ou apenas documentação, evitando bloqueios desnecessários em tarefas administrativas.

### ✨ O que ele faz
1.  **Validação de Fluxo:** Bloqueia merges diretos de `dev` para `production` ou features direto para `staging`.
2.  **Smart Diff:** Ignora validações estritas se a mudança for apenas em arquivos de documentação (ex: `README.md`).
3.  **Feedback Visual:** Posta um comentário claro no PR explicando por que o fluxo foi aprovado ou rejeitado.
4.  **Integração:** Expõe `outputs` para você encadear outras actions (como auto-sync ou deploys).

---

## 🚦 Regras do Fluxo

Esta Action impõe a seguinte esteira de promoção:

```mermaid
graph LR
    DEV[Development] -->|✅ Permitido| STG[Staging]
    STG[Staging] -->|✅ Permitido| MAIN[Production]
    
    FEAT[Feature/*] -.->|🚫 Bloqueado| MAIN
    DEV -.->|🚫 Bloqueado| MAIN
````

  * **Production** (main/master) só aceita merges vindos de **Staging**.
  * **Staging** (homolog/release) só aceita merges vindos de **Development**.
  * Qualquer outra combinação gera um alerta de bloqueio.

-----

## 📦 Instalação

Adicione este job no topo do seu workflow de Pull Request.

### Permissões Necessárias

Como esta action posta comentários no PR, você precisa conceder permissão de escrita.

```yaml
permissions:
  pull-requests: write
  contents: read
```

### Exemplo de Workflow

```yaml
name: "Governance Check"

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  flow-guard:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
    steps:
      - name: Validate Branch Flow
        id: check
        uses: Malnati/flow-check@v1.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

-----

## ⛓️ Exemplo Avançado (Job Chaining)

O poder real desta action está em usar seus **Outputs** para controlar a execução de jobs subsequentes (ex: só rodar testes pesados ou sync se o fluxo for válido).

```yaml
jobs:
  # 1. O Porteiro
  governance:
    runs-on: ubuntu-latest
    outputs:
      allowed: ${{ steps.guard.outputs.allowed }}
      has_code: ${{ steps.guard.outputs.has_code }}
    steps:
      - name: Run Guard
        id: guard
        uses: Malnati/flow-check@v1.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

  # 2. Job Pesado (Só roda se permitido e tiver código)
  heavy-tests:
    needs: governance
    if: ${{ needs.governance.outputs.allowed == 'true' && needs.governance.outputs.has_code == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Rodando testes de integração..."
```

-----

## ⚙️ Inputs & Outputs

### Inputs

| Input | Obrigatório | Descrição |
| :--- | :---: | :--- |
| `token` | **Sim** | Token do GitHub (`secrets.GITHUB_TOKEN`) para ler diffs e postar comentários. |

### Outputs

Valores retornados para uso em steps seguintes (`${{ steps.id.outputs.nome }}`).

| Output | Tipo | Descrição |
| :--- | :---: | :--- |
| `allowed` | `true/false` | Define se o fluxo de branches respeita as regras. |
| `has_code` | `true/false` | Define se há alterações em arquivos de código (ignora docs). |
| `head_branch` | String | Nome da branch de origem (ex: `feature/login`). |
| `base_branch` | String | Nome da branch de destino (ex: `develop`). |

-----

<div align="center">

<sub>Security & Governance by <a href="https://github.com/Malnati">Ricardo Malnati</a>.</sub>

</div>
