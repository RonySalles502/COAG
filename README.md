# COAG/DPE-RN — Distribuição de Demandas

Sistema interno da Coordenadoria de Análise e Gestão da Defensoria Pública do Estado do Rio Grande do Norte para distribuição equitativa de demandas (ETP, TR, Mapa de Risco, aditivos) entre fiscais primários, com base em matriz de complexidade SCT e algoritmo de menor carga ativa.

Aplicação **standalone** (HTML + CSS + JS em arquivo único), sem dependências externas, com persistência via `localStorage` e proteções multicamada contra perda de dados.

---

## Hospedagem no GitHub Pages

### Passos

1. Crie um repositório no GitHub (público ou privado — privado exige plano pago para Pages).
2. Faça upload de todos os arquivos desta pasta para a raiz do repositório.
3. Em **Settings → Pages**, selecione:
   - Source: `Deploy from a branch`
   - Branch: `main` (ou `master`) — pasta `/ (root)`
4. Aguarde 1–2 minutos. O endereço será `https://SEU-USUARIO.github.io/NOME-DO-REPO/`.

O arquivo `.nojekyll` evita que o GitHub Pages tente processar a aplicação como blog Jekyll.

### Estrutura de arquivos esperada

```
.
├── index.html              # Aplicação principal
├── manifest.webmanifest    # Manifesto PWA
├── sw.js                   # Service Worker (cache offline)
├── icons/
│   ├── icon-192.png
│   └── icon-512.png
├── .nojekyll               # Sinaliza ao Pages para servir como estático puro
└── README.md
```

---

## Proteções contra perda de dados

A aplicação salva tudo no **navegador do usuário** (não em servidor). Quatro camadas reduzem o risco:

| Camada | O que faz | Limitação |
|---|---|---|
| **Persistência declarada** (`navigator.storage.persist`) | Solicita ao navegador que não apague o storage em limpezas automáticas de cache. | Não impede limpeza explícita pelo usuário ("Limpar dados deste site"). |
| **Snapshots rotativos** | Salva cópia diária do estado dos últimos 7 dias no próprio localStorage. Permite reverter alterações acidentais. | Vai junto se o usuário apagar o storage. |
| **Backup automático em pasta** | Salva um JSON na pasta do computador escolhida pelo usuário a cada alteração (debounce 5s). Único método que **sobrevive a limpeza total do navegador**. | **Funciona apenas em Chromium** (Chrome, Edge, Opera, Brave). Firefox e Safari não suportam — usuários nesses navegadores ficam limitados ao export/import manual. |
| **Detecção de quota** | Avisa quando o uso do storage se aproxima do limite. | Apenas informativo. |

A aba **Backup** da aplicação mostra o status de cada camada e permite configurar cada uma. Um banner vermelho no topo aparece quando há risco real (ex.: sem backup em pasta configurado).

### Recomendação de uso

1. **Na primeira abertura**: ativar persistência e configurar pasta de backup automático.
2. **Recomendado**: pasta dentro de OneDrive, Google Drive ou Dropbox — assim os JSONs ficam sincronizados na nuvem automaticamente.
3. **Após reinicialização do navegador**: a permissão da pasta precisa ser **reconfirmada** (Chrome exige interação). A aba Backup sinaliza isso.

---

## Uso como PWA (Progressive Web App)

A aplicação é instalável como app nativo. Em Chrome/Edge, aparece um botão de instalação na barra de endereço ou em "três pontinhos → Instalar app". Uma vez instalada:

- Abre em janela própria, sem abas do navegador
- Funciona offline (depois da primeira carga, o app shell fica cacheado)
- Ícone fica disponível como qualquer aplicativo

Os **dados** continuam no localStorage e na pasta de backup configurada — não são cacheados pelo Service Worker.

---

## Compatibilidade entre navegadores

| Recurso | Chrome / Edge / Opera | Firefox | Safari |
|---|---|---|---|
| Aplicação básica | ✅ | ✅ | ✅ |
| `navigator.storage.persist` | ✅ | ✅ | parcial |
| Snapshots em localStorage | ✅ | ✅ | ✅ |
| Backup automático em pasta | ✅ | ❌ | ❌ |
| Instalável como PWA | ✅ | ✅ (Android) | ✅ (iOS, com restrições) |
| Service Worker offline | ✅ | ✅ | ✅ |

**Para uso institucional, recomenda-se Chrome ou Edge** — são os únicos que entregam todas as proteções.

---

## Uso multi-usuário

O `localStorage` é **por navegador**. Cada servidor da equipe que abrir a página tem seu próprio estado isolado. Para colaboração em tempo real, é necessário backend (não incluso).

Alternativas práticas:

- **Uma pessoa como fonte de verdade** (a coordenadora ou alguém designado): mantém o estado autoritativo, exporta JSON ao fim do dia e distribui aos demais, que importam pela aba Backup.
- **Pasta de backup compartilhada** via OneDrive/Drive: cada usuário aponta sua pasta de backup para o mesmo diretório sincronizado. Última gravação vence — risco de conflito se dois editarem ao mesmo tempo.
- **Migração para backend**: Google Sheets API, Airtable, Notion ou SQLite+servidor. Requer desenvolvimento adicional.

---

## Estrutura dos dados

JSON exportável tem o formato:

```json
{
  "processos": [
    {
      "id": "p_xxx",
      "sei": "000110000061.000007/2026-06",
      "objeto": "...",
      "tipoDemanda": "completo" | "apenas_tr" | "aditivo",
      "dimensoes": { "A": 5, "B": 2, "C": 1, "D": 0, "E": 1 },
      "score": 9,
      "nivelCodigo": "N4",
      "sctBase": 8,
      "modificador": 1.0,
      "sctFinal": 8,
      "fiscalElaboracao": "Liza" | "Thiago" | "Lucas" | "Matheus" | null,
      "status": "elaboracao" | "revisao_matheus" | "aprovacao_kerolaine" | "aprovado" | "bloqueado_externo",
      "aguardandoExterno": "CEAP" | "CTI" | "CCSC" | "Outro" | null,
      "observacoes": "...",
      "criadoEm": 1234567890,
      "ultimaAtualizacao": 1234567890,
      "entrouFilaEm": 1234567890,
      "historico": [{ "ts": 1234567890, "evento": "..." }]
    }
  ],
  "config": {
    "limiarSuplencia": 12,
    "slaMatheus": 2,
    "slaKerolaine": 2
  }
}
```

---

## Matriz SCT (resumo)

| Dimensão | Faixa | Critério |
|---|---|---|
| A. Natureza | 0–5 | Dedicação exclusiva MO=5 · TI/engenharia=4 · Contínuo=3 · Simples=2 · Bens comuns=0 |
| B. Escopo | 0–2 | Aquisição=0 · TR padrão=1 · TR complexo=2 |
| C. Apoio externo | 0–1 | Nenhum=0 · CTI/CEAP/CCSC=1 |
| D. Prazo | 0–1 | Regular=0 · Urgente=1 |
| E. Valor | 0–1 | Até R$100k=0 · Acima=1 |

**Conversão:** Score = A+B+C+D+E (0–10) → N1 (0–2) = 1 SCT · N2 (3–5) = 2 SCT · N3 (6–8) = 4 SCT · N4 (9–10) = 8 SCT

**Modificador por tipo:** Completo ×1,0 · Apenas TR ×0,6 · Aditivo/Apostilamento = SCT fixo 0,3

---

## Limitações conhecidas

- Pontuação inicial dos processos legados é estimativa baseada em descrição — precisa de revisão pela coordenadoria antes de operar produtivamente.
- O sistema só reflete a carga real se for alimentado prospectivamente. Planilhas históricas registram resultado, não esforço.
- Aprovação exclusiva da Coordenadora é gargalo de processo — o sistema o torna visível via SLA, não o resolve.
- Multi-usuário concorrente exige backend (ver seção acima).
