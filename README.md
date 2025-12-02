# flow-check
Fluxo reutilizável de validação de PRs (dev → staging → main) para repositórios GitHub. Detecta mudanças de código, verifica se o fluxo de promoção de branches é respeitado e expõe saídas (has_code, allowed, base_branch, head_branch) para orquestrar auto-sync, PMD, Prettier e outros workflows.
