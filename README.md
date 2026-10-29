# API ClickUp DWG → SVG

API FastAPI que recebe um webhook do ClickUp, localiza o primeiro anexo `.dwg` da tarefa, envia o arquivo para a AutoDWG Conversion API, aguarda a conversão e anexa o `.svg` de volta à mesma tarefa.

## Fluxo

```text
ClickUp → POST /webhook/clickup → API própria → AutoDWG → SVG → anexo na tarefa ClickUp
```

Não é necessário instalar AutoCAD, Aspose.CAD ou outro binário CAD.

## 1. Configuração local

```bash
cd dwg-clickup-api
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Edite `.env` e preencha:

```env
CLICKUP_API_TOKEN=pk_...
AUTODWG_API_KEY=...
WEBHOOK_SECRET=uma-chave-longa-e-secreta
```

O `WEBHOOK_SECRET` é criado por você. Use uma senha longa, sem espaços, e não a publique.

Execute:

```bash
./start.sh
```

Verifique:

```bash
curl http://localhost:8000/health
```

Resposta esperada:

```json
{"status":"ok"}
```

## 2. Endpoint do ClickUp

A URL pública final será:

```text
https://SEU-DOMINIO/webhook/clickup
```

O ClickUp deve enviar um POST JSON e um dos headers abaixo:

```text
x-api-key: WEBHOOK_SECRET
```

ou, para compatibilidade com a configuração anterior do Make:

```text
x-make-apikey: WEBHOOK_SECRET
```

Exemplo de payload mínimo:

```json
{"task_id":"86abc123"}
```

O payload real do ClickUp também pode usar `task.id`; a API aceita ambos.

## 3. Configuração da automação no ClickUp

1. Abra a List que contém as tarefas.
2. Abra **Automations**.
3. Crie uma regra com o gatilho **Status altered**.
4. Use, por exemplo:
   - De: `RECEBIDO DWG`
   - Para: `CONVERTER PARA SVG`
5. Na ação escolha **Call webhook / Chamar webhook**.
6. Use a URL pública da API:
   `https://SEU-DOMINIO/webhook/clickup`
7. Em headers, adicione:
   - `Content-Type: application/json`
   - `x-api-key: o mesmo valor de WEBHOOK_SECRET`
8. Não coloque tokens do ClickUp ou AutoDWG no ClickUp.
9. Salve e teste com uma tarefa que já possua um `.dwg` anexado.

Como a API busca os anexos depois de receber o ID da tarefa, o DWG pode já estar anexado antes da mudança de status.

## 4. Hospedagem

A API precisa de uma URL pública HTTPS. Pode ser hospedada em qualquer serviço que aceite Python/Docker, por exemplo:

- Cloud Run;
- Render;
- Railway;
- uma VPS.

Use o `Dockerfile` incluído. Configure as variáveis do `.env` como **secrets** no provedor; não faça commit do arquivo `.env`.

Com Docker local:

```bash
docker build -t dwg-clickup-api .
docker run --env-file .env -p 8000:8000 dwg-clickup-api
```

Para produção, use HTTPS do próprio provedor. Não exponha a porta sem TLS diretamente na internet.

## 5. Comportamento e limites

- Aceita o primeiro anexo com extensão `.dwg`.
- Limita o download ao valor de `MAX_FILE_MB` (padrão: 5 MB).
- Usa `output_format=svg` na AutoDWG.
- Consulta o status a cada `POLL_INTERVAL_SECONDS`.
- Interrompe após `MAX_POLL_ATTEMPTS` consultas.
- Retorna HTTP 202 rapidamente e processa a conversão em background.

## 6. Segurança

- Nunca publique `.env`.
- Troque o `WEBHOOK_SECRET` se ele aparecer em uma captura de tela.
- Troque o token do ClickUp ou a chave AutoDWG se forem expostos.
- Comece com DWGs não confidenciais.
- Não registre o conteúdo binário dos arquivos.
- Confirme a política de retenção da AutoDWG antes de enviar desenhos sob NDA.

## 7. Erros comuns

- `401 Invalid webhook key`: o header não corresponde a `WEBHOOK_SECRET`.
- `400 Could not find task_id`: ajuste o corpo enviado pelo ClickUp ou envie `task_id`/`task.id`.
- `No .dwg attachment found`: a tarefa não possui anexo `.dwg` ou a API não conseguiu obter a URL.
- `401` no ClickUp: token pessoal inválido ou ausente.
- `FileTooLarge` na AutoDWG: reduza o arquivo ou altere o plano/limite permitido.
