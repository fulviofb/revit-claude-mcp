# Revit + Claude Code via pyRevit 7 (MCP)

Passo a passo para conectar o **Autodesk Revit** ao **Claude Code** usando o Agent Runtime (MCP) do **pyRevit 7**. Com isso o Claude lê o modelo (quantitativos, tabelas, parâmetros, folhas) e faz alterações que você aprova dentro do próprio Revit.

Testado em 27/10/2026: Windows 11, Revit 2027.3, pyRevit 7.0.0.

## Como usar

O guia foi escrito para o **Claude Code** te conduzir. É ele que roda os comandos no seu computador; o Claude do navegador não consegue.

**Opção 1: como skill (recomendado)**

1. Baixe este repositório (botão verde **Code › Download ZIP**) ou clone:
   ```bash
   git clone https://github.com/fulviofb/revit-claude-mcp
   ```
2. Copie a pasta `revit-mcp-setup` para `C:\Users\<seu usuário>\.claude\skills\`.
3. Abra o Claude Code e peça: *"quero conectar o Revit ao Claude"*.

**Opção 2: colando o guia**

Abra uma sessão do Claude Code, anexe ou cole o arquivo [`revit-mcp-setup/SKILL.md`](revit-mcp-setup/SKILL.md) e escreva *"siga este guia"*.

## O que o guia cobre

- Comparação das opções de MCP para Revit e por que o pyRevit 7
- Instalação do pyRevit (o que marcar, quais avisos aceitar)
- Ligar o Agent Runtime e registrar no Claude Code
- Teste de leitura: quantitativo de tubos por sistema e diâmetro
- Problemas encontrados e como resolver (acentos no IronPython, PATH, sessão nova)
- Cuidados: cópia do arquivo, responsabilidade técnica, dados de clientes, licença educacional

## Avisos

- Trabalhe numa **cópia** do .rvt nos primeiros usos.
- A responsabilidade técnica (ART/RRT) continua sendo do profissional.
- O Agent Runtime do pyRevit ainda é **experimental**.
- Este guia não tem vínculo com a Autodesk nem com o pyRevit.

## Licença

MIT
