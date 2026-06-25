[Arquitetura_Run_of_Show.md](https://github.com/user-attachments/files/29338025/Arquitetura_Run_of_Show.md)
# Run of Show — Arquitetura, Fluxo de Dados e Mapeamento de Campos

Documento de referência do sistema Run of Show (ROS): editor, viewer, e a forma como os dados circulam entre o Google Sheets, o Google Apps Script, a Firebase, o GitHub e o Netlify.

---

## 1. Papel de cada sistema

| Sistema | Papel |
|---|---|
| **Google Sheets** (`RunOfShow_DB`) | Fonte de verdade do alinhamento. Guarda todo o conteúdo das cues de forma persistente. |
| **Code.gs** (Google Apps Script) | A ponte. Serve o editor, lê e escreve no Sheets, expõe os endpoints que o viewer consome e empurra sinais para a Firebase. |
| **Firebase Realtime Database** (`rosrise-46f05`) | Só o canal de tempo real. Guarda o ponteiro do cue ao vivo e as mensagens. Não guarda o alinhamento. |
| **GitHub** (`brunodomingos-maker/ros-viewer`) | Repositório onde vive o código-fonte do `live.html`. |
| **Netlify** (`sunny-paletas-f1c5f6.netlify.app`) | Aloja o `live.html` como ficheiro estático, sem login. É o que a equipa abre. |

Ideia-chave: o **alinhamento** (conteúdo) viaja sempre por Code.gs ↔ Sheets. O **cue e as mensagens** (tempo real) viajam por Firebase, do editor direto para o viewer, sem tocar no Sheets.

---

## 2. Fluxo de escrita (editor)

1. Editas o alinhamento no editor e gravas. O `google.script.run` chama o `saveRows` no Code.gs.
2. O Code.gs escreve o JSON do alinhamento no Sheets (partido em pedaços na coluna A do separador `show_<id>`), atualiza a cache, e empurra para a Firebase um sinal com o `dataVer` novo (`fbSyncFromPayload`).
3. Quando carregas em Next ou Prev ao vivo, o editor escreve diretamente na Firebase (`pushLive`): `liveCue`, `seq` incremental e `ts`.
4. As mensagens flash (Stand by, Coffee break) vão para `shows/<show>/live/msg`.

```
Editor --saveRows--> Code.gs --grava chunks--> Google Sheets
Editor --saveRows--> Code.gs --fbSync: dataVer--> Firebase
Editor --pushLive: liveCue, seq, ts--> Firebase
Editor --msg--> Firebase (shows/<show>/live/msg)
```

---

## 3. Fluxo de leitura (viewer)

1. O viewer (`live.html` no Netlify) arranca e faz um pedido JSONP ao Code.gs: `?action=getData&show=...`. O Code.gs devolve o alinhamento todo (da cache ou do Sheets) mais o cue atual e a versão.
2. O viewer subscreve `shows/<show>/live` na Firebase e recebe em tempo real o cue, o dia e as mensagens. Move o cue e mostra os flashes, sem voltar a passar pelo Code.gs.
3. Se o `dataVer` mudar (sinal de que o alinhamento foi alterado), o viewer chama de novo o `getData` para reler.
4. Se a Firebase ficar offline mais de 8 segundos, o viewer faz polling do `getData` a cada 3 segundos como rede de segurança.

```
Viewer --getData (JSONP)--> Code.gs --loadRows--> Google Sheets (ou cache)
Viewer <--tempo real: cue, dia, msg-- Firebase (shows/<show>/live)
Netlify --aloja--> Viewer   |   GitHub --código--> Netlify
```

---

## 4. Modelo do alinhamento (o JSON em `data`)

### Topo do objeto

| campo | tipo | o que é |
|---|---|---|
| `days` | array | lista de dias |
| `screenNames` | array[4] | nomes dos 4 ecrãs |
| `evtData` | objeto | `{name, date, venue, caller}` |
| `liveId` | string | id do cue ao vivo |
| `currentDay` | número | índice do dia ativo |

### Dia (`days[i]`)

| campo | o que é |
|---|---|
| `id` | id do dia |
| `label` | nome do dia |
| `rows` | linhas (items e secções) |

### Linha do tipo item (`type: 'item'`)

| campo | o que é | coluna no editor |
|---|---|---|
| `type` | `'item'` | — |
| `id` | id único da cue | — |
| `name` | nome da cue | Item |
| `start` | hora de início HH:MM | Start |
| `dur` | duração em minutos | Dur. |
| `speaker` | orador | (sob o nome) |
| `audio` | ação de som | Audio |
| `slides` | ação de micro | **Micro** |
| `video` | ação de vídeo | Video |
| `lights` | ação de luz | Lights |
| `furniture` | ação de palco | **Palco / Stage** |
| `screens[4]` | conteúdo dos 4 ecrãs | Screen 1 a 4 |
| `screenUrls[4]` | links dos ecrãs | — |
| `screenImgs[4]` | imagens dos ecrãs | — |
| `depts[]` | etiquetas dos departamentos | chips (SND/LGT/VID/CAM/STG) |
| `notes` | notas | Notes |
| `status` | estado da cue | — |

### Linha do tipo secção (`type: 'sec'`)

| campo | o que é |
|---|---|
| `type` | `'sec'` |
| `id` | id |
| `name` | nome da secção |
| `color` | cor da secção |

> Atenção aos dois nomes herdados: a coluna **Micro** guarda no campo `slides`, e a coluna **Palco** guarda no campo `furniture`. O conteúdo é o esperado, só o nome do campo é antigo.

---

## 5. Nó da Firebase (`shows/<show>/live`)

| campo | o que é |
|---|---|
| `liveCue` | id do cue ao vivo (espelha o `liveId`) |
| `liveDay` | índice do dia ativo (espelha o `currentDay`) |
| `seq` | contador incremental, ordena as escritas e ignora as fora de ordem |
| `ts` | timestamp da escrita |
| `dataVer` | versão dos dados, sinal para o viewer reler o `getData` |
| `msg.text` | texto da mensagem flash |
| `msg.ts` | timestamp da mensagem (ignora mensagens antigas ao abrir) |

A Firebase tem leitura e escrita públicas apenas no nó `/live`.

---

## 6. Resposta do `getData` (JSONP)

| campo | o que é |
|---|---|
| `ok` | sucesso |
| `data` | o JSON do alinhamento, em string (o objeto do topo da secção 4) |
| `name`, `date` | nome e data do evento |
| `liveId` / `liveCue` | cue ao vivo (o segundo é alias do primeiro) |
| `currentDay` / `liveDay` | dia ativo (alias) |
| `screenNames` | nomes dos ecrãs |
| `ver` | versão dos dados |

---

## 7. Departamento → campo (vista por função no viewer)

| pílula no viewer | lê do campo |
|---|---|
| Som | `audio` |
| Micro | `slides` |
| Vídeo | `video` |
| Luz | `lights` |
| Palco | `furniture` |
| Ecrãs | `screens[]` (juntos com ` · `) |

As chips de departamento (`depts[]`) usam os códigos SND, LGT, VID, CAM, STG. O link do viewer aceita `role=som,luz` ou `role=SND,LGT` para abrir já na vista filtrada (até 3 departamentos em simultâneo). Aceita ainda `view=table` ou `view=caller` para escolher o modo de arranque.

---

## 8. Como o Sheets guarda (separador `show_<id>`)

| sítio | conteúdo |
|---|---|
| linha 1, coluna A | `json` (cabeçalho) |
| linha 3, coluna A | JSON com `{venue, caller}` |
| linha 4 em diante, coluna A | o alinhamento todo, partido em pedaços (DATA_START = 4) |
| linha 5, coluna B | JSON das pastas do Drive |

Separadores de apoio:

| separador | conteúdo |
|---|---|
| `__meta__` | `show_id`, `name`, `date`, `created_at` de cada show |
| `__auth__` | senhas e tokens: `pw_<show>`, `token_<show>`, `admin_password` |

---

## 9. Identificadores do projeto

| item | valor |
|---|---|
| Firebase projeto | `rosrise-46f05` (região US) |
| Firebase databaseURL | `https://rosrise-46f05-default-rtdb.firebaseio.com` |
| Netlify (viewer) | `https://sunny-paletas-f1c5f6.netlify.app` |
| GitHub (viewer) | `brunodomingos-maker/ros-viewer` |
| URL do viewer | `<netlify>/#gas=<exec_url>&show=<ID>&role=<departamentos>` |

> O URL `/exec` do Apps Script é específico da organização (`/a/macros/rise.pt/s/...`). Tira-o sempre pelo botão Share do editor (`getDeployUrl`), nunca o construas à mão.

---

## 10. Notas operacionais

- **Sempre que mexes no `Code.gs` ou no `RunOfShow.html`**: Deploy → Gerir implementações → editar a ativa → Versão = Nova versão → Implementar. Não cries uma implementação nova (muda o `/exec`).
- **Sempre que mexes no `live.html`**: republica no Netlify (ou via GitHub se ligado).
- O viewer do Netlify não precisa de login Google. O `?view=1` servido pelo GAS pede login, por isso a equipa usa o link do Netlify.
- Resumo de uma alteração ao alinhamento: editas, o Code.gs grava no Sheets e levanta o `dataVer` na Firebase, o viewer vê o `dataVer` mudar e relê o `getData`.
- Resumo de um Next: o editor escreve `liveCue` na Firebase, o viewer recebe na hora e acende a cue, sem passar pelo Sheets.
