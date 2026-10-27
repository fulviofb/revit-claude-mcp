---
name: revit-mcp-setup
description: Guia passo a passo para conectar o Autodesk Revit (2022–2027) ao Claude Code usando o Agent Runtime (MCP) do pyRevit 7. Use quando o usuário quiser instalar pyRevit, ligar o MCP do Revit, conectar o Claude ao Revit, extrair quantitativos/tabelas do modelo ou automatizar o Revit com IA.
---

# Conectar o Revit ao Claude Code (pyRevit 7 Agent Runtime)

Você (Claude) vai guiar um engenheiro, em português, a conectar o Revit ao Claude Code. Este roteiro foi executado com sucesso em 27/10/2026 num Windows 11 com Revit 2027.3 e pyRevit 7.0.0. O usuário provavelmente conhece AutoCAD e tem pouca prática com Revit: explique sem jargão de programação.

## Por que este caminho

Avaliamos as opções antes de escolher esta:

| Opção | Lê | Altera | Observação |
|---|---|---|---|
| **pyRevit 7 Agent Runtime** (escolhida) | sim | sim, com aprovação dentro do Revit | Oficial do pyRevit, gratuito; o Claude escreve scripts Python contra a API do Revit |
| MCP oficial Autodesk (Revit 2027.2+) | sim | não | Bom complemento só leitura; baixa em manage.autodesk.com › Revit 2027 › Extensions |
| mcp-servers-for-revit (C#) | sim | sim | Release v1.0.0 só vai até Revit 2026 |
| mcp-server-for-revit-python | sim | sim | Usa pyRevit Routes; ficou redundante com o runtime do pyRevit 7 |

Só testamos no Revit 2027; os testes de CI do pyRevit 7 cobrem 2022–2027. Para versão mais antiga, confira as notas da release do pyRevit antes de seguir.

## Etapa 0 — Diagnóstico (você faz)

Rode no PowerShell e leia o resultado:

```powershell
Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\* | ? DisplayName -match "Revit|pyRevit" | Select DisplayName,DisplayVersion,InstallLocation
claude --version
```

Confirme a versão do Revit e se o pyRevit já existe. O Revit pode estar instalado fora de `C:\Program Files` (no nosso caso estava em `D:\Autodesk 2027`), então confie no registro, não na pasta.

## Etapa 1 — Instalar o pyRevit 7 (o usuário faz)

Instalador é .exe: o usuário baixa e roda, não você.

1. Fechar o Revit.
2. Baixar a versão mais recente em https://github.com/pyrevitlabs/pyRevit/releases — o arquivo `pyRevit_<versão>_signed.exe` (por usuário, sem admin). Use `_admin_signed.exe` só se a conta não puder instalar programas. Para achar o link exato: `gh api repos/pyrevitlabs/pyRevit/releases/latest --jq '.assets[].name'`.
3. Na tela **Select Components**: manter **Core Tools** (obrigatório) e **Extended Extensions**. Developer e Tutorials não são necessários.
4. Abrir o Revit. Aparece "Segurança – Complemento assinado" para o `pyRevitLoader.dll`, assinado por **Jean-Marc Couffin** (mantenedor do pyRevit) → clicar **Sempre carregar**. Pode aparecer mais de um aviso; mesma resposta.
5. A aba **pyRevit** deve surgir na faixa de opções.

## Etapa 2 — Ligar o runtime e registrar no Claude Code (você faz)

O instalador pode ter usado uma pasta personalizada e o PATH da sessão atual pode não estar atualizado. Ache o executável:

```powershell
$env:Path = [Environment]::GetEnvironmentVariable('Path','User') + ';' + [Environment]::GetEnvironmentVariable('Path','Machine')
(Get-Command pyrevit).Source
pyrevit --version
```

Depois (use o caminho completo encontrado acima se `pyrevit` não estiver no PATH):

```powershell
pyrevit attached                 # deve mostrar o Revit do usuário anexado
pyrevit configs agent enable     # liga o Agent Runtime (vem desligado)
pyrevit mcp install claude       # registra o servidor "pyrevit" no Claude Code (escopo user, ~/.claude.json)
claude mcp list                  # deve listar "pyrevit ... ✓ Connected"
```

Peça ao usuário: **aba pyRevit › Reload** (ou reiniciar o Revit) e **abrir um projeto**. Sugestão de teste: os arquivos de amostra na tela inicial do Revit (ex.: "Amostra proj. hidráulico").

## Etapa 3 — Verificar (você faz)

```powershell
pyrevit agent status          # esperado: "Revit 20xx ... responding", policy: ask
pyrevit agent context         # documento ativo, vista, níveis, engines disponíveis
```

## Etapa 4 — Primeiro teste, só leitura (você faz)

Salve este script num arquivo temporário (ex.: `teste_tubos.py`). Ele soma tubulação por sistema e diâmetro — útil para orçamento. Em projeto sem tubos, troque por paredes ou ambientes.

```python
from Autodesk.Revit.DB.Plumbing import Pipe

pipes = DB.FilteredElementCollector(doc).OfClass(Pipe).WhereElementIsNotElementType()
totais = {}
for p in pipes:
    sis = p.get_Parameter(DB.BuiltInParameter.RBS_SYSTEM_CLASSIFICATION_PARAM).AsString() or "?"
    diam_mm = round(p.Diameter * 304.8)
    comp_m = p.get_Parameter(DB.BuiltInParameter.CURVE_ELEM_LENGTH).AsDouble() * 0.3048
    chave = "%s | %d mm" % (sis, diam_mm)
    t = totais.setdefault(chave, [0, 0.0])
    t[0] += 1
    t[1] += comp_m

result = {
    "total_tubos": pipes.GetElementCount(),
    "por_sistema_diametro": sorted(
        [{"grupo": k, "qtd": v[0], "comprimento_m": round(v[1], 1)} for k, v in totais.items()],
        key=lambda x: -x["comprimento_m"])[:15],
}
```

```powershell
pyrevit agent run teste_tubos.py --mode=query --engine=cpython --title="Teste quantitativo tubos"
```

Resultado de referência (Snowdon Towers Sample Plumbing): 3.051 tubos; "Água fria doméstica | 25 mm" = 670 un / 1.632 m. Apresente ao usuário em tabela.

O runner injeta `doc`, `uidoc`, `app`, `DB`, `UI`; o script atribui `result`. Unidades internas do Revit são pés (×0,3048 para metros).

## Etapa 5 — Usar no dia a dia

1. O usuário abre uma **sessão nova** do Claude Code — as ferramentas MCP (`get_context`, `inspect_elements`, `run_query`, `run_modify`, `capture_view`, `lookup_revit_api`, `get_skill`…) só aparecem em sessões iniciadas depois do registro.
2. Na sessão nova, leia primeiro a skill `revit-scripting` via `get_skill`, como o servidor instrui.
3. Alterações (`run_modify`) abrem uma janela **dentro do Revit** com os elementos destacados: **Keep** grava como um único Ctrl+Z; **Discard** desfaz. Use `dry_run=true` para prever antes.

## Armadilhas que encontramos

- **Acentos quebram o IronPython.** O motor padrão (IronPython 2.7) falha com `ResultSerializationError ... can't decode byte 0xc1` ao devolver nomes como "Água fria". Use sempre `engine=cpython` (CPython 3.12, já vem com o pyRevit) em `run_query`/`run_modify`.
- **Licença educacional.** Se a barra de título do Revit mostrar "EDUCACIONAL (NÃO COMERCIAL)", avise: serve para aprender, não para projeto pago; arquivos salvos ficam marcados e a marca aparece na impressão e contamina arquivos comerciais que recebam esses dados.
- **PATH desatualizado** logo após instalar: recarregue o PATH como na Etapa 2 ou use o caminho completo do `pyrevit.exe`.
- **Ferramentas não aparecem no chat**: faltou sessão nova do Claude Code, ou o Revit não foi recarregado depois do `agent enable`.

## Cuidados para passar ao usuário

- Trabalhar sempre numa **cópia** do .rvt nos primeiros usos.
- A responsabilidade técnica (ART/RRT) continua do engenheiro.
- O .rvt fica na máquina, mas os dados consultados vão para o Claude — evitar dados sensíveis de clientes.
- A IA é boa em dados, tabelas, parâmetros, auditoria e rotinas de prancha; ainda falha em geometria complexa.
- Nunca expor portas do pyRevit na rede (o Agent Runtime usa pipe local; se ligar o servidor Routes, manter em 127.0.0.1).
