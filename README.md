Com base na evolução que construímos (separação de responsabilidades, outputs JSON ricos, suporte a Sticky Comments e validação dinâmica), o README antigo do `flow-check` está obsoleto.

Abaixo está o **novo `README.md`** profissional, focado na nova arquitetura da **Branch Flow Guard Pro**. Ele documenta os inputs de configuração, o formato do JSON e fornece o "Workflow de Ouro" combinando as duas Actions.

````markdown
# 🛡️ Branch Flow Guard Pro

[![GitHub Release](https://img.shields.io/github/v/release/Malnati/branch-flow-guard?style=for-the-badge&color=purple)](https://github.com/Malnati/branch-flow-guard/releases)
[![License](https://img.shields.io/github/license/Malnati/branch-flow-guard?style=for-the-badge&color=blue)](LICENSE)

** Governança de código inteligente e modular para GitHub Actions.**

O **Branch Flow Guard Pro** é um analisador lógico de Pull Requests. Diferente de linters tradicionais, ele valida a **matemática do Git Flow** do seu projeto, garantindo que o ciclo de vida do software seja respeitado (ex: impedir merge de `feature` direto em `production`).

> 💡 **Nota de Arquitetura:** Esta Action adere ao princípio de responsabilidade única. Ela **não posta comentários**. Ela analisa o fluxo e retorna um objeto JSON rico (com vereditos e orientações) para ser consumido por outras actions (como `Malnati/pr-comment`).

---

## 🚀 Funcionalidades

* **🛡️ Governança Configurável:** Defina quais branches são Produção, Staging e Desenvolvimento via inputs.
* **🧠 Orientação Dinâmica:** Gera mensagens de erro educativas, explicando exatamente para qual branch o desenvolvedor deveria ter apontado a PR.
* **⚡ Smart Bypass:** Detecta se a PR contém código fonte ou apenas documentação (opcional).
* **JSON Output:** Retorna um payload completo para integrações avançadas (Slack, Teams, Dashboards).

---

## 📦 Inputs

Todos os inputs são opcionais (possuem defaults sensatos), mas totalmente configuráveis.

| Input | Descrição | Padrão |
| :--- | :--- | :--- |
| `token` | **Obrigatório**. Token do GitHub para ler arquivos da PR. | - |
| `production_branches` | Lista de branches de Nível 1 (Produção). | `main, master, prod` |
| `staging_branches` | Lista de branches de Nível 2 (Homologação). | `staging, homol, release` |
| `development_branches` | Lista de branches de Nível 3 (Dev). | `dev, develop, development` |
| `output_file` | Nome do arquivo JSON gerado no workspace. | `flow-compliance.json` |

---

## 📤 Outputs

A action disponibiliza os resultados de duas formas:

### 1. Variável de Output (`steps.id.outputs.result`)
Um JSON stringificado contendo toda a análise.

### 2. Arquivo Físico (`flow-compliance.json`)
Ideal para upload de artefatos ou depuração.

#### Exemplo do JSON Gerado
```json
{
  "version": "1.2.0",
  "timestamp": "2023-10-27T10:00:00Z",
  "compliance": {
    "allowed": false,
    "violation_code": "PROD_VIOLATION"
  },
  "context": {
    "head_branch": "feature/login",
    "base_branch": "main"
  },
  "ui": {
    "message_md": "🚫 **Produção** (main) requer origem em **Staging**.",
    "guidance_md": "Para mergear em `main`, a branch de origem deve ser uma destas: **[staging, release]**.",
    "color": "#d73a49"
  }
}
````

-----

## 🛠️ Exemplo de Uso (Workflow Completo)

Este é o padrão recomendado: **Validação (Guard)** + **Notificação (Sticky Comment)** + **Bloqueio (Enforcement)**.

Crie o arquivo `.github/workflows/branch-flow.yml`:

```yaml
name: "Branch Governance"

on:
  pull_request:
    types: [opened, synchronize, reopened, edited]

permissions:
  contents: read        # Ler config do repo
  pull-requests: write  # Postar comentários

jobs:
  governance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      # 1. Analisa o Fluxo (Define as Regras)
      - name: Branch Flow Guard
        id: flow
        uses: Malnati/branch-flow-guard@v1.2.1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # Personalize suas branches aqui:
          production_branches: "main"
          staging_branches: "homol, staging"
          development_branches: "develop"

      # 2. Carrega o JSON gerado
      - name: Load Compliance Data
        id: data
        run: |
          {
            echo "json_content<<EOF"
            cat flow-compliance.json
            echo "EOF"
          } >> "$GITHUB_OUTPUT"

      # 3. Notifica o Usuário (Comentário Inteligente/Sticky)
      - name: Post Governance Comment
        uses: Malnati/pr-comment@v6.1.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          pr_number: ${{ github.event.pull_request.number }}
          template_path: .github/workflows/pr-comment-branch-flow.md # Crie este arquivo no seu repo
          message_id: "branch-flow-guard-status" # Garante que o bot atualize o mesmo comentário
          
          header_title: "🛡️ Branch Flow Guard"
          header_subject: "Validacao de Fluxo"
          header_actor: "github-actions[bot]"
          
          # Renderiza a tabela de status baseada no JSON da etapa 1
          body_message: |
            <div align="center">
            
            | 🚦 Status | 🛫 Origem | 🛬 Destino |
            | :---: | :---: | :---: |
            | **${{ fromJson(steps.data.outputs.json_content).ui.message_md }}** | `${{ fromJson(steps.data.outputs.json_content).context.head_branch }}` | `${{ fromJson(steps.data.outputs.json_content).context.base_branch }}` |
            
            </div>
          
          # Exibe o veredito (apenas se bloquear)
          footer_result: >-
             ${{ !fromJson(steps.data.outputs.json_content).compliance.allowed && 
             format('⛔ **Bloqueado** (Código: {0})', fromJson(steps.data.outputs.json_content).compliance.violation_code) || 
             '' }}
          
          # Exibe a orientação educativa gerada pela Action
          footer_advise: ${{ fromJson(steps.data.outputs.json_content).ui.guidance_md }}

      # 4. Bloqueia o Merge se necessário
      - name: Enforce Governance
        if: ${{ !fromJson(steps.data.outputs.json_content).compliance.allowed }}
        run: |
          # Exibe a mensagem de erro no log do Actions
          GUIDANCE="${{ fromJson(steps.data.outputs.json_content).ui.guidance_md }}"
          CLEAN_GUIDANCE=$(echo "$GUIDANCE" | sed 's/\*\*//g' | sed 's/`//g')
          
          echo "::error title=Branch Flow Violation::$CLEAN_GUIDANCE"
          exit 1
```

-----

## 🎨 Template Markdown Recomendado

Para o passo de comentário funcionar visualmente bem, crie o arquivo `.github/workflows/pr-comment-branch-flow.md` no seu repositório:

```markdown
## ${TITLE}

> [!NOTE]
> **Fluxo de Referência (Governance):**
>
> 1. `✨ Feature/Fix` &rarr; 🛠️ **Development** _(develop)_
> 2. 🛠️ **Development** &rarr; 🧪 **Staging** _(homol)_
> 3. 🧪 **Staging** &rarr; 🚀 **Production** _(main)_
>
> *Siga estritamente a ordem sequencial acima.*

${BODY_MESSAGE}

${FOOTER_BLOCK}
```

-----

\<div align="center"\>
\<sub\>Developed by \<a href="https://github.com/Malnati"\>Ricardo Malnati\</a\>\</sub\>
\</div\>

```
```
