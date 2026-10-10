# Plano: versão Android do BunkrDownloader

Este documento descreve como transformar o BunkrDownloader (CLI em Python) em um
aplicativo Android nativo.

## 1. O que o projeto faz hoje (escopo a portar)

| Módulo atual | Responsabilidade | Destino no Android |
|---|---|---|
| `src/misc/url_utils.py` | Normaliza URL, identifica álbum (`/a/`) vs arquivo (`/f/`, `/v/`, `/i/`), fallback de domínio (`bunkr.cr`) | `domain/UrlResolver.kt` |
| `src/crawlers/crawler_utils.py` | Faz scraping do álbum (paginação `?page=N`), lista de itens, nome do arquivo (inclui decodificação de e-mail Cloudflare) | `data/scraper/AlbumScraper.kt` (Jsoup) |
| `src/crawlers/api_utils.py` | Extrai `jsCDN` do JS inline, chama `DOWNLOAD_API` (POST) e `BUNKR_API` (assinatura → `token` + `ex`) | `data/api/BunkrApi.kt` (OkHttp/Retrofit) |
| `src/crawlers/date_utils.py` | Data do item → `mtime` do arquivo | `data/scraper/DateParser.kt` |
| `src/downloaders/media_downloader.py` | Download com retry/backoff, detecção de servidor offline | `download/MediaDownloadWorker.kt` |
| `src/downloaders/download_utils.py` | Download paralelo por *byte ranges*, `.partN`, retomada, merge | `download/ChunkedDownloader.kt` |
| `src/downloaders/album_downloader.py` | Orquestra o álbum, reserva nomes, re-tenta falhas | `download/AlbumDownloadWorker.kt` |
| `src/managers/state_manager.py` | `.bunkr_state.json` para retomar álbuns | Banco Room (`DownloadItem`, `Album`) |
| `src/managers/rate_limiter.py` | Limite de banda global | `download/RateLimiter.kt` (token bucket compartilhado) |
| `src/misc/bunkr_utils.py` | Status dos servidores (status.bunkr.ru) | `data/StatusRepository.kt` |
| `src/config.py` + `bunkr.toml` | Configurações | Tela de Configurações + DataStore |
| `--ignore` / `--include` / `--clean-name` / `--dry-run` | Filtros e pré-visualização | Filtros na tela do álbum + seleção manual de itens |
| `main.py` + `URLs.txt` | Lote de URLs | Fila de downloads (colar várias URLs / importar .txt) |
| `rich` (UI de terminal) | Progresso | Notificações + tela de fila em Compose |

São ~4.400 linhas de Python, mas a lógica específica do Bunkr (scraping + assinatura)
cabe em ~500. O resto é infraestrutura de download, que no Android é reescrita de
qualquer forma (WorkManager, notificações, armazenamento).

## 2. Decisão de arquitetura

### Opção recomendada: reescrita nativa em Kotlin

- **UI:** Jetpack Compose + Material 3.
- **Rede:** OkHttp (+ Retrofit opcional para as duas APIs JSON); `Jsoup` substitui o BeautifulSoup.
- **Downloads em segundo plano:** WorkManager com `setForeground()` (foreground service tipo `dataSync`), sobrevive ao app fechado e a reinícios.
- **Persistência:** Room (fila, estado por item, partes concluídas) + DataStore (configurações).
- **Arquivos:** Storage Access Framework (usuário escolhe a pasta) ou `MediaStore` para `Download/Bunkr`.
- **DI:** Hilt. **Coroutines/Flow** para estado de progresso.
- **minSdk 26**, target/compile SDK atual.

Por quê: APK pequeno, integração correta com o ciclo de vida do Android (Doze,
restrições de serviço em segundo plano, permissões de notificação), e a lógica a portar
é pequena e bem isolada.

### Alternativa considerada: embutir o Python (Chaquopy)

Reaproveitaria `src/crawlers/*` quase sem mudanças, mas: APK de 30–50 MB, `aiohttp`
exige wheels nativos, `rich`/`sys.exit`/caminhos relativos teriam de ser removidos, e o
controle de background ainda precisaria ser em Kotlin. Útil só como protótipo rápido —
**não recomendada** para o produto.

## 3. Estrutura do projeto

```
android/
  app/src/main/java/.../bunkr/
    ui/            # Compose: Home, AlbumDetail, Queue, Settings
    domain/        # UrlResolver, modelos (Album, MediaItem, DownloadState), filtros
    data/
      scraper/     # AlbumScraper, ItemPageParser, DateParser, CfEmailDecoder
      api/         # BunkrApi (sign + download API), StatusRepository
      db/          # Room: entidades, DAOs
      prefs/       # DataStore
    download/      # Workers, ChunkedDownloader, RateLimiter, FileNamer
  app/src/test/    # testes JVM com HTML de fixtures
```

O app fica em `android/` dentro deste mesmo repositório, para acompanhar mudanças do
upstream no Python (os seletores/endpoints mudam com frequência e o upstream costuma
corrigir primeiro).

## 4. Fases

### Fase 0 — Preparação (0,5 semana)
- Criar projeto Gradle em `android/` (Kotlin DSL, version catalog).
- Capturar fixtures HTML reais (página de álbum, álbum paginado, item imagem/vídeo/zip,
  e-mail Cloudflare) para testes.
- CI: GitHub Actions com `./gradlew lint test assembleDebug`, mantendo o `pylint.yml` existente.

### Fase 1 — Núcleo (portar a lógica do Bunkr) (1,5 semana)
1. `UrlResolver`: normalização, tipo de URL, fallback de domínio.
2. `AlbumScraper`: itens + paginação + datas.
3. `ItemPageParser`: nome do arquivo (com `decrypt_cf_email` e correção latin1→utf8), `jsCDN`, `data-file-id`.
4. `BunkrApi`: POST `DOWNLOAD_API` (`Accept-Encoding: gzip, deflate`), GET `BUNKR_API` → `url?token=…&ex=…`, retry com backoff exponencial.
5. `FileNamer`: lógica de `_format_item_filename`, `--clean-name`, caracteres inválidos, truncamento (120), nomes duplicados.
6. Testes unitários para cada um, comparando com a saída da versão Python nas mesmas fixtures.

**Critério de saída:** dado um link de álbum, o app lista itens com nome e URL assinada correta.

### Fase 2 — Motor de download (2 semanas)
1. Download simples com stream, headers (`User-Agent`, `Referer`), retry/backoff, tratamento de 403 (DDoS-Guard), 404, 5xx, servidor offline.
2. Download paralelo por ranges (>64 MB, 4 conexões, unidades de 4–64 MB) com arquivos `.partN` e retomada — estado das partes guardado no Room.
3. Rate limiter global compartilhado.
4. `AlbumDownloadWorker` + `MediaDownloadWorker` com limite de concorrência, re-tentativa das falhas no fim, `mtime` a partir da data do item.
5. Renovação de URL assinada quando expirar (`ex`) durante retomadas.
6. Verificação de espaço livre (equivalente ao `MIN_DISK_SPACE`).

**Critério de saída:** baixar um álbum grande, matar o app no meio e retomar sem re-baixar.

### Fase 3 — Interface (1,5 semana)
- **Home:** colar URL(s), botão de colar da área de transferência, importar `.txt`.
- **Intent filter / Share target:** abrir links `bunkr.*` e receber "Compartilhar" do navegador.
- **Detalhe do álbum (dry-run):** pré-visualização com nome, tamanho, data; filtros incluir/ignorar; seleção manual.
- **Fila:** progresso por item e geral, pausar/retomar/cancelar, reprocessar falhas.
- **Notificações:** progresso contínuo + resumo final (equivalente ao `summary_manager`).
- **Configurações:** pasta de destino (SAF), subpasta por álbum, conexões, retries, limite de banda, concorrência, só Wi-Fi.
- **Log:** tela com o `session.log` (exportável).

### Fase 4 — Robustez e polimento (1 semana)
- Restrições do WorkManager (rede não medida, bateria), comportamento em Doze.
- Permissão `POST_NOTIFICATIONS` (Android 13+), tipo de foreground service (Android 14+).
- Tela de status dos servidores Bunkr.
- Tema claro/escuro, i18n (pt-BR e en).
- Testes instrumentados básicos de UI.

### Fase 5 — Distribuição (0,5 semana)
- Build release assinado via GitHub Actions (keystore em *secrets*), publicado em GitHub Releases.
- Verificação de atualização pelo próprio app (consulta à última release).
- **Google Play não é recomendado:** apps de download de hosts de terceiros costumam ser
  rejeitados pelas políticas da loja. Distribuir por APK (sideload) ou F-Droid/Obtainium.

**Estimativa total:** ~7 semanas para uma pessoa (MVP utilizável ao fim da Fase 2 + parte da 3, ~4 semanas).

## 5. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Bunkr muda HTML, domínios ou endpoints de assinatura com frequência | Isolar seletores/endpoints em um único `BunkrConfig`; testes com fixtures; acompanhar commits do upstream (`git remote add upstream https://github.com/Lysagxra/BunkrDownloader`) |
| DDoS-Guard/Cloudflare bloqueia requisições (403) | Headers iguais aos do Python; backoff; fallback de domínio; opcionalmente um WebView para obter cookies de desafio |
| Sistema mata o processo em segundo plano | WorkManager + foreground service; estado persistido em Room |
| Armazenamento com escopo (Android 10+) | SAF/MediaStore; nada de `WRITE_EXTERNAL_STORAGE` |
| URL assinada expira durante download longo | Re-assinar antes de cada retomada |
| Licença | O projeto é **GPL-3.0**: o app derivado, se distribuído, deve ser GPL-3.0 com código-fonte disponível. Manter `LICENSE` e créditos ao autor original |

## 6. Próximos passos imediatos

1. Resolver a questão do repositório privado (ver abaixo).
2. Fase 0: gerar o esqueleto do projeto Android em `android/` e capturar fixtures.
3. Fase 1, item 1–4, com testes — primeira PR.

### Sobre o repositório privado

O fork `OPandaBR/BunkrDownloader` existe, mas é **público**, e o GitHub não permite
tornar privado um fork de repositório público. Para ter uma cópia privada:

1. Criar um repositório vazio e privado (ex.: `OPandaBR/BunkrDownloader-Android`) em
   github.com/new.
2. Espelhar o código:
   ```bash
   git clone --bare https://github.com/Lysagxra/BunkrDownloader.git
   cd BunkrDownloader.git
   git push --mirror https://github.com/OPandaBR/BunkrDownloader-Android.git
   ```
3. No clone de trabalho, adicionar o original como `upstream` para puxar correções:
   `git remote add upstream https://github.com/Lysagxra/BunkrDownloader.git`.
4. Opcional: apagar o fork público depois de migrar.
