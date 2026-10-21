# OmniRoute — Docker / konteyner derin analiz raporu

**Durum:** Tamamlandı, salt-okunur.
**Tam rapor:** `/private/var/folders/d6/x4vcs4f16qbd43dgp1nlm8pm0000gn/T/opencode/docker-reviewer-report.md` (633 satır)
**Repo durumu:** `git status --porcelain` 9 önceden var olan değişiklik gösteriyor; bu inceleme sırasında **yeni repo değişikliği üretilmedi**. Docker/Podman çalıştırılmadı, imaj build edilmedi, Trivy/Fly/Podman doğrulaması yapılmadı.

---

## 1. Yönetici özeti

Docker bu repoda tek bir `Dockerfile` değil; **yayınlanan imajlar + Compose profilleri + sidecar'lar + CI/CD + CLI otomasyonu + testler + dokümanlar** olarak geniş bir ürün yüzeyi.

Olumlu taraf:

- Ana `Dockerfile` ciddi biçimde sertleştirilmiş: `npm ci --ignore-scripts`, lockfile zorunluluğu, non-root `node`, `REQUIRE_API_KEY=true`, loopback-first varsayılan, Trivy CRITICAL kapısı ve SBOM.
- `contrib/vps/compose.yaml` ve `docker/devin-bridge/compose.yml` iyi sertleştirme örnekleri: `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, `mem_limit`, `read_only`, loopback bind.
- CDP proxy'lerinde token kontrolü ve fail-closed davranış var; VNC/CDP portları host'ta loopback'e bağlanıyor.

Ana riskler:

- **Bun imajı** (`Dockerfile.bun`) Node imajının güvenlik duruşunu taşımıyor: `REQUIRE_API_KEY=true` yok, lockfile yok, `--ignore-scripts` yok.
- **`cli` profili** host Docker socket'ini ve repo kökünü `rw` mount ediyor; bu, host root eşdeğeri bir risk.
- **VNC browser** konteyneri sertleştirilmemiş; `linuxserver/chromium:latest` kullanıyor ve görünürde VNC/noVNC parolası yok.
- Compose varsayılanı `.env` üzerinden `REQUIRE_API_KEY=false`; host portları loopback olsa da **aynı compose ağındaki sibling container'lar** anonim `/v1` proxy'ye erişebilir.
- CI yalnızca taban Node imajını smoke/Trivy/SBOM ile kapsıyor; `-web`, `-bun`, `-web-bun` varyantları taranmıyor.

**Sentez önceliği:** 0 kritik, 8 yüksek, 10 orta, 6 düşük, 16 bayat/kırık madde.

“Kritik 0” = varsayılan uzaktan, kimlik doğrulamasız RCE yok. Ancak `cli` profili ve VNC gibi opt-in yüzeyler açıldığında risk yükseliyor.

---

## 2. Kapsam ve yöntem

- `Dockerfile`, `Dockerfile.bun`, dört Compose dosyası, `docker/`, `contrib/podman`, `contrib/vps`, CI/publish hattı, scriptler, kaynak kod, testler, dokümanlar ve i18n tarandı.
- Hariç tutuldu: `node_modules`, `dist/node_modules`, `electron/node_modules`, `.next`, `.claude/worktrees`, vendor dizinleri.
- Yalnızca statik analiz: dosya okuma, `rg`, `git status`, `command -v`. Docker/Podman komutu çalıştırılmadı.
- Runtime gerektiren iddialar “statik olarak muhtemel” olarak işaretlendi.

### Reviewer taslağına göre düzeltmeler

- Testlerin tamamı metin tabanlı değil: **3 test gerçekten konteyner çalıştırıyor**:
- `tests/security/test-docker-hardening.sh` — Docker build/run.
- `tests/security/test-cli-runtime.sh` — Docker build/run/exec.
- `tests/integration/liveContainerHarness.ts` — Podman; `RUN_LIVE_WIRE_CAPTURE=1` ile.
- Apple Container sağlayıcısında `--cap-drop ALL` ve `no-new-privileges` **var**; eksik olan WSL ve OrbStack.
- `bin/cli/commands/redis.mjs`, `fly.toml` ve `bin/rollback.sh` Docker bağımlılıkları taslakta eksikti; eklendi.
- `STA-16` (Redis opsiyonel) büyük olasılıkla yanlış; Compose'ta Redis profilsiz ve her zaman tanımlı.
- `Dockerfile.bun` içinde `REQUIRE_API_KEY=true` olmaması ve CI'nin yalnızca taban imajı taraması taslakta yoktu; eklendi.

---

## 3. Envanter

### Kök seviye

- `Dockerfile` — 403 satır; çok aşamalı Node imajı.
- `Dockerfile.bun` — 166 satır; Bun imajı.
- `docker-compose.yml` — 440 satır; ana stack, profiller ve sidecar'lar.
- `docker-compose.prod.yml` — 118 satır; prod benzeri stack.
- `docker-compose.selfhost.yml` — 79 satır; self-host stack.
- `docker-compose.ollama-local.yml` — host'ta çalışan Ollama için overlay.
- `.dockerignore`, `.trivyignore`, `fly.toml`.

### `docker/`

- `docker/chatgpt-web-codex-browser/` — Playwright tabanlı ChatGPT Web Codex browser imajı + CDP proxy.
- `docker/vnc-browser/chromium/` — `linuxserver/chromium` tabanlı VNC login browser + CDP bridge.
- `docker/devin-bridge/` — Devin bridge test harness; Compose + Dockerfile + network guard.

### `contrib/`

- `contrib/podman/` — Podman Quadlet dosyaları: `omniroute.container`, `omniroute-redis.container`, network/volume.
- `contrib/vps/compose.yaml` — en iyi sertleştirme örneği.
- `contrib/windows/` — container değil; Windows başlatma scriptleri.

### CI/CD

- `.github/workflows/docker-publish.yml` — 766 satır; çok mimarili build, manifest, smoke, Trivy, SBOM, Docker Hub açıklaması.
- `.github/workflows/ci.yml:227` — hadolint `docker run` ile.
- `.github/workflows/semgrep.yml:18` — GitHub Actions job container.
- `.github/workflows/deploy-vps.yml:4-6` — Docker publish ile tetikleniyor; `:60` ise `npm install -g omniroute@latest` çalıştırıyor.

### Kaynak kod

- `src/lib/vncSession/service.ts` — VNC browser konteynerini `docker` CLI ile başlatıyor.
- `src/lib/skills/containerProvider.ts` — Docker, Apple Container, WSL, OrbStack, Podman sandbox sağlayıcıları.
- `src/lib/system/autoUpdate.ts` / `dockerUpdateTrigger.ts` — Docker image/compose/trigger güncelleme modları.
- `src/shared/utils/containerEnv.ts` — Docker/Podman/containerd/kubepods algılama.
- `src/lib/startup/nonLoopbackApiKeyGuard.ts` — non-loopback + keyless uyarısı.
- `src/lib/apiBridgeServer.ts`, `src/server/ws/liveServer.ts` — `API_HOST` / `LIVE_WS_HOST` bind davranışı.
- `bin/cli/commands/redis.mjs` — Podman/Docker ile Redis başlatma.
- `bin/cli/utils/environment.mjs` — `/.dockerenv` ve cgroup algılama.
- `bin/rollback.sh` — Docker ve npm rollback yöntemleri.
- `src/lib/container.ts` — **yanlış pozitif**: DI container, Docker değil.
- `open-sse/utils/composerToolCalls.ts` — **yanlış pozitif**: Docker Compose değil.

### Testler

- `tests` altında Docker/Podman geçen **159 dosya** var; çoğu metin/sözleşme testi.
- Gerçekten konteyner çalıştıran 3 test yukarıda; hiçbiri CI'da değil.
- `tests/unit/issue-13679-container-posture-require-api-key.test.ts` yalnızca `Dockerfile` ve `fly.toml` kontrol ediyor; `Dockerfile.bun` kapsam dışı.
- `tests/unit/vnc-session-docker-args.test.ts` yalnızca ağ/token kontrol ediyor; `--cap-drop` / `no-new-privileges` yok.
- `tests/unit/skills-builtins-sandbox.test.ts` yalnızca Docker argümanlarını kontrol ediyor; Apple/WSL/OrbStack yok.

### Dokümanlar ve i18n

- `docs/guides/DOCKER_GUIDE.md` — 840 satır; ana Docker rehberi.
- `docs/getting-started/SELF_HOST_GUIDE.md`, `docs/guides/LOCAL_OLLAMA_DOCKER.md`, `docs/providers/CURSOR-DOCKER.md`, `docs/providers/ZED-DOCKER.md`, `docs/ops/VM_DEPLOYMENT_GUIDE.md`, `docs/ops/FLY_IO_DEPLOYMENT_GUIDE.md`, `docs/guides/UNINSTALL.md`.
- `src/i18n/messages/tr.json:10658` — `deployDockerTitle` değeri `"Liman işçisi"`; Docker için yanlış çeviri. Birçok dilde benzer transliterasyon/çeviri var.
- `scripts/i18n/glossary/protected-terms.json` içinde `"Docker"` korumalı terim olarak tanımlı; buna rağmen başlık çevirisi bozulmuş.

---

## 4. Mimari

### Node imajı

`Dockerfile` aşamaları:

1. **base** — `node:26-trixie-slim`, apt upgrade, npm CVE overlay, non-root `node`.
2. **builder** — `npm ci --ignore-scripts`, `better-sqlite3` rebuild, `npm run build`, standalone çıktı.
3. **runner-base** — yalın çalışma imajı; `check-permissions.sh`, healthcheck, `USER node`, `ENV REQUIRE_API_KEY=true`, `DATA_DIR=/app/data`.
4. **runner-web** — Playwright/Chromium eklenmiş web-cookie sağlayıcı imajı; root ile kurulum, sonra `USER node`.
5. **runner-cli** — Docker CLI/Compose ve 5 agent CLI'si (`codex`, `claude-code`, `droid`, `openclaw`, `qodercli`) eklenmiş imaj; sonra `USER node`.

### Bun imajı

- `oven/bun:1.4.3-slim` tabanlı.
- `bun install --include=optional`, native `wreq-js` ve `bun:sqlite` smoke testleri var.
- `runner-base` ve `runner-web` aşamaları var; `USER bun`.
- Node imajındaki `REQUIRE_API_KEY=true`, `check-permissions.sh`, runtime base-path patch'i, `OMNIROUTE_BUILD_MEMORY_MB` ARG'si ve `NODE_OPTIONS` heap ayarı **yok**.
- `bun.lock` repo'da yok; `.dockerignore:104` ayrıca `bun.lock`'u dışlıyor. `bun install` bu yüzden tekrarlanabilir değil.

### Compose

Ana `docker-compose.yml` profilleri:

- `base` — `omniroute-base`, yalın sunucu.
- `web` — `omniroute-web`, Chromium/Playwright.
- `cli` — `omniroute-cli`, agent CLI'leri + Docker socket.
- `host` — host CLI config mount'ları.
- `memory` — Qdrant.
- `bifrost` — Bifrost sidecar.
- `cliproxyapi` — CLIProxyAPI sidecar.
- `codex-app-server` — Codex app-server sidecar.

Redis profilsiz ve her zaman tanımlı. Ayrıca `chatgpt-web-codex-browser` ve `codex-app-server` sidecar'ları var.

### Yayın hattı

`docker-publish.yml`:

- `runner-base`, `runner-web`, `runner-base` (Bun), `runner-web` (Bun) çok mimarili build.
- Manifest oluşturma; `-web`, `-bun`, `-web-bun` etiketleri.
- `:VERSION` taban imajı için smoke test.
- Yalnızca taban imaj için Trivy ve SBOM.
- `latest` promosyonu; Bun manifestleri opsiyonel.
- Docker Hub README güncellemesi.

### Dağıtım tüketicileri

Docker'a bağlı ürün yüzeyleri:

- Yayınlanan Docker Hub/GHCR imajları.
- Fly.io: `fly.toml:9` boş `[build]` nedeniyle `Dockerfile` kullanılıyor; `REQUIRE_API_KEY=true`.
- VPS: `contrib/vps/compose.yaml`; `deploy-vps.yml` ise npm ile kuruyor ama Docker publish ile tetikleniyor.
- Podman: `contrib/podman` Quadlet.
- VNC browser login: `src/lib/vncSession` Docker CLI'ye bağımlı.
- ChatGPT Web Codex: Compose sidecar + CDP.
- Devin bridge: Docker Compose test harness.
- CLI Redis: `bin/cli/commands/redis.mjs` Podman/Docker.
- Auto-update: Docker image/compose/trigger modları.
- Rollback: `bin/rollback.sh --method docker`.

### Ağ ve güven sınırları

- Host portları çoğunlukla `127.0.0.1`; iyi.
- Ancak container içinde uygulama `0.0.0.0` dinliyor; loopback host bind'i **sibling container'ları korumuyor**.
- Redis, Qdrant, Bifrost, CLIProxyAPI ve uygulama varsayılan compose ağında birlikte.
- `chatgpt-web-codex-net` yalnızca browser'ı diğerlerinden ayırıyor; `omniroute-web` hem bu ağda hem default ağda.
- `cliproxyapi` credential taşıyan bir sidecar; loopback host bind'i container-to-container erişimi engellemiyor.
- `codex-app-server` capability token ile korunuyor; iyi.
- `vnc-browser` CDP bridge token ile korunuyor; ancak noVNC UI için parola yok.

---

## 5. Güvenlik bulguları

### Yüksek

#### Y1. Compose varsayılanı keyless + düz ağ

**Kanıt:**

- `.env.example:448` — `REQUIRE_API_KEY=false`.
- `docker-compose.yml:39` — `env_file: .env`; `environment` içinde `REQUIRE_API_KEY` yok.
- `Dockerfile:274` — `ENV REQUIRE_API_KEY=true`; ancak Compose `.env` değeri imaj ENV'sini eziyor.
- `docker-compose.yml:113-115` — host portları loopback.
- `Dockerfile` — uygulama container içinde `HOSTNAME=0.0.0.0`.
- `src/lib/startup/nonLoopbackApiKeyGuard.ts` — yalnızca uyarıyor, engellemiyor.

**Etki:** Host'tan doğrudan erişim yok; fakat aynı compose ağındaki Redis/Qdrant/Bifrost/CLIProxyAPI/browser gibi bir sibling ele geçirilirse, `/v1` proxy'sine API key olmadan erişilebilir. `cliproxyapi` credential'lı olduğu için bu, yatay hareket riski.

**Öneri:** Compose'da `REQUIRE_API_KEY: ${REQUIRE_API_KEY:-true}` kullan; `.env.example` içinde container bölümünü `true` yap veya ayrı `.env.docker.example` oluştur. Sidecar'ları ayrı internal network'lere ayır; Redis'e `requirepass`, Qdrant/Bifrost/CLIProxyAPI'ye auth ekle.

#### Y2. Bun imajı keyless

**Kanıt:**

- `Dockerfile.bun` içinde `ENV REQUIRE_API_KEY=true` yok.
- `Dockerfile.bun:79` — `HOSTNAME=0.0.0.0`.
- `docker-publish.yml:542-543` — `-bun` / `-web-bun` manifestleri yayınlanıyor.
- `tests/unit/issue-13679-container-posture-require-api-key.test.ts:34-40` — yalnızca `Dockerfile` ve `fly.toml` kontrol ediliyor; Bun kapsam dışı.

**Etki:** `docker run -p 20128:20128 diegosouzapw/omniroute:latest-bun` gibi bir kullanımda uygulama dünyaya açık ve keyless olabilir. Node imajında bu risk #13679 ile kapatılmış; Bun imajında açık kalmış.

**Öneri:** `Dockerfile.bun`'a `ENV REQUIRE_API_KEY=true` ekle; testi her iki Dockerfile'ı kapsayacak şekilde parametreleştir. Bun imajı “best-effort” kalacaksa, `-bun` etiketlerini yayınlama veya açıkça “desteklenmiyor” olarak işaretle.

#### Y3. `cli` profili: Docker socket + repo kökü `rw`

**Kanıt:**

- `docker-compose.yml:216` — `/var/run/docker.sock:/var/run/docker.sock`.
- `docker-compose.yml:218` — `${AUTO_UPDATE_HOST_REPO_DIR:-.}:/host-repo:rw`.
- `Dockerfile` `runner-cli` — `docker.io`, `docker-compose` ve agent CLI'leri kuruyor.
- `src/lib/system/autoUpdate.ts` — Docker socket'e bağımlı `docker-compose` modu.
- `src/lib/system/dockerUpdateTrigger.ts` — socket'siz `docker-image` / `docker-trigger` alternatifi sunuyor.

**Etki:** Bu profil açıldığında uygulama veya içindeki herhangi bir agent, host Docker daemon'u üzerinden host root eşdeğeri yetki kazanabilir. Repo kökünün `rw` mount'u ile host dosyalarını değiştirebilir. Opt-in olsa da “tek kullanıcılı workstation” varsayımına dayanıyor.

**Öneri:** `docker.sock` mount'unu kaldır; `DOCKER_UPDATE_TRIGGER_FILE` tabanlı socket'siz modu varsayılan yap. Socket gerekliyse ayrı bir override dosyasına taşı, repo mount'unu `ro` yap ve açıkça `--profile cli` + dokümantasyon uyarısıyla sınırla.

#### Y4. VNC browser konteyneri sertleştirilmemiş

**Kanıt:**

- `src/lib/vncSession/service.ts:173-201` — `docker run` argümanlarında `--cap-drop`, `--security-opt`, `--user`, `--pids-limit`, `--memory` yok.
- `docker/vnc-browser/chromium/Dockerfile:21` — `FROM linuxserver/chromium:latest`.
- `Dockerfile:26-29` — noVNC UI 3000, CDP 9222/9223.
- `service.ts` — yalnızca `CHROME_CLI` ve `CDP_BRIDGE_TOKEN` set ediliyor; `PASSWORD` / `CUSTOM_USER` yok.
- `service.ts:191` — VNC UI `127.0.0.1` üzerinde yayınlanıyor; iyi, ancak çok kullanıcılı host'ta yerel kullanıcılar erişebilir.
- `docker/vnc-browser/chromium/cdp-bridge.py` — token substring kontrolü; `timingSafeEqual` değil.

**Etki:** VNC/noVNC UI loopback'te parolasız olabilir; yerel bir süreç/kullanıcı oturum açmış browser'a bağlanıp credential/cookie çalabilir. CDP token'ı yalnızca CDP'yi korur, UI'yı korumaz. Ayrıca mutable `latest` taban imajı ve eksik container sertleştirmesi supply-chain/escape riskini artırır.

**Öneri:** `linuxserver/chromium` için `CUSTOM_USER` + `PASSWORD` veya per-session token'lı bir reverse proxy ekle; `--cap-drop ALL`, `--security-opt no-new-privileges`, `--pids-limit`, `--memory` ekle; `linuxserver/chromium` imajını digest'e pinle; healthcheck ekle; CDP bridge'de sabit-zamanlı karşılaştırma kullan.

#### Y5. Bun build supply-chain / tekrarlanabilirlik

**Kanıt:**

- `Dockerfile.bun:23` — `COPY package.json bun.lock* ...`; ancak `.dockerignore:104` `bun.lock`'u dışlıyor.
- `Dockerfile.bun:32` — `bun install --include=optional`; `--ignore-scripts` veya `--frozen-lockfile` yok.
- `Dockerfile.bun` — `OMNIROUTE_BUILD_MEMORY_MB` ARG'si yok; `docker-publish.yml:319` bu argümanı geçiyor.
- `Dockerfile.bun` — `oven/bun:1.4.3-slim`; proje pinli Bun sürümü 1.4.2.
- `docker-publish.yml:311,339` — Bun build'leri `continue-on-error: true`.

**Etki:** Build sırasında lifecycle script'leri çalışabilir; lockfile olmadığı için bağımlılıklar sürüm kayabilir; Bun sürümü test edilenden farklı olabilir; build bellek argümanı sessizce yok sayılır. Bu, yayınlanan `-bun` imajları için tekrarlanabilirlik ve supply-chain riski.

**Öneri:** `bun.lock`'u repo'ya ekle ve `.dockerignore`'dan çıkar; `bun install --frozen-lockfile --ignore-scripts` kullan; lifecycle script gerekiyorsa allowlist uygula; `OMNIROUTE_BUILD_MEMORY_MB` ARG'sini ekle; Bun sürümünü proje pinine hizala; Bun imajı için smoke test ekle.

#### Y6. `docker-compose.prod.yml` `runner-cli` kullanıyor

**Kanıt:**

- `docker-compose.prod.yml:14-26` — başlıkta varsayılanın `runner-base` olduğu söyleniyor.
- `docker-compose.prod.yml:53` — `target: runner-cli`.
- `Dockerfile` `runner-cli` — Docker CLI, Compose ve 5 agent CLI'si içeriyor.

**Etki:** Prod uygulaması için gereksiz Docker CLI/agent CLI'leri ve daha büyük saldırı yüzeyi. Başlık ile uygulama çelişiyor; operatör yanlış imajı bekleyebilir.

**Öneri:** `runner-base` kullan; auto-update gerekiyorsa `docker-trigger` moduna geç. `runner-cli` yalnızca `cli` profiline ait olsun. Başlığı ve dokümanı hizala.

#### Y7. CI yalnızca taban imajı tarıyor/test ediyor

**Kanıt:**

- `docker-publish.yml:575` — smoke test `${IMAGE_NAME}:${VERSION}` yalnızca taban imaj.
- `:632` — SBOM yalnızca taban imaj.
- `:653,674` — Trivy yalnızca taban imaj.
- `:542-543` — `-web`, `-bun`, `-web-bun` manifestleri yayınlanıyor.
- `:311,339` — Bun build'leri opsiyonel.

**Etki:** Chromium/Playwright içeren `-web` imajı ve farklı tabanlı `-bun` imajları smoke/Trivy/SBOM kapsamı dışında. `-web` imajında bir CRITICAL CVE veya başlatma hatası release'i bloklamaz.

**Öneri:** Her yayınlanan varyant için en azından `docker run` + healthcheck smoke testi çalıştır; Trivy/SBOM'u `-web`, `-bun`, `-web-bun` için de üret. Bun “best-effort” ise, bu durumu manifest ve dokümanda açıkça belirt.

#### Y8. Supply-chain pinleme tutarsız

**Kanıt:**

- `docker-publish.yml` içinde bazı action'lar SHA pinli (`aquasecurity/trivy-action`, bazı `build-push` adımları), bazıları major tag pinli:
- `actions/checkout@v7`
- `docker/login-action@v4.6.0`
- `docker/setup-buildx-action@v4`
- `docker/build-push-action@v7`
- `anchore/sbom-action@v0`
- `peter-evans/dockerhub-description@v5`
- Taban imajlar digest'e sabit değil: `node:26-trixie-slim`, `oven/bun:1.4.3-slim`, `linuxserver/chromium:latest`, Redis/Qdrant/Bifrost/CLIProxyAPI tag'leri.
- `ci.yml:227` hadolint'i digest ile sabitliyor; iyi örnek.

**Etki:** Tag'ler mutable; supply-chain saldırısı veya sessiz sürüm kayması riski. Özellikle `linuxserver/chromium:latest` en kötü örnek.

**Öneri:** Tüm action'ları SHA'ya pinle; taban imajları digest'e pinle; Renovate/Dependabot ile güncelle; `linuxserver/chromium` için özel, digest-pinli bir imaj kullan.

---

### Orta

#### O1. Ana Compose stack'inde container sertleştirmesi yok

`docker-compose.yml` içinde `cap_drop`, `security_opt`, `pids_limit`, `mem_limit`, `read_only` yok. `contrib/vps/compose.yaml:46-54` ve `docker/devin-bridge/compose.yml:34-38` altın standart. Ana stack için de en azından `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, `mem_limit` eklenmeli.

#### O2. Prod'da CDP ağı izolasyonu kaybolmuş

Kök `docker-compose.yml:188-189` `chatgpt-web-codex-browser` için `chatgpt-web-codex-net` kullanıyor. `docker-compose.prod.yml:94-108` bu ağı kullanmıyor; browser default ağa düşüyor. Prod'da da aynı izolasyonu uygula.

#### O3. Aynı `container_name: omniroute` dört kez

`docker-compose.yml:100,125,196,225` — base, web, cli, host servisleri aynı container adını kullanıyor. Profiller birlikte açılırsa çakışma/Compose hatası riski var. `container_name` kaldır veya profile özel yap.

#### O4. `API_HOST` / `LIVE_WS_HOST` loopback vs yayınlanan portlar

- `docker-compose.yml:46` — `API_HOST=127.0.0.1`; `:114` — 20129 publish.
- `docker-compose.yml:48` — `LIVE_WS_HOST=127.0.0.1`; `:115` — 20132 publish.
- `src/lib/apiBridgeServer.ts:310` — API bridge `API_HOST` üzerinde dinliyor.
- `src/server/ws/liveServer.ts:188` — live WS `LIVE_WS_HOST` üzerinde dinliyor.

Container içinde loopback'e bağlanan bir servis, host'a publish edilen porttan normalde erişilemez. Bu, statik olarak işlevsiz port eşlemesi gibi görünüyor. Ya `0.0.0.0` bind + host loopback publish kullan, ya da bu portları publish etme. Runtime doğrulaması gerekli.

#### O5. Podman Quadlet tüm arayüzlere açıyor

`contrib/podman/omniroute.container:17-18` — `PublishPort=20128:20128`, `20129:20129`; host IP kısıtı yok. `:29` — `API_HOST=0.0.0.0`. `:32` — `REQUIRE_API_KEY=true`; iyi. Ancak `DropCapability`, `NoNewPrivileges`, `ReadOnly`, `PidsLimit`, `Memory` yok. Rootful Podman'de portlar LAN'a açılır. `127.0.0.1:20128:20128` ve sertleştirme ekle.

#### O6. `docker-compose.ollama-local.yml` 0.0.0.0 + `:latest`

`:26` — `diegosouzapw/omniroute:latest`; `:36` — `0.0.0.0:20128`; `env_file` yok. Geliştirme overlay'i olduğu açıkça belirtilmeli; production için loopback + pinli sürüm + API key/env dosyası gerekli. `REQUIRE_API_KEY` görünürde yok; Node imajının `true` varsayılanına güveniyor.

#### O7. CLI Redis launcher eski ve sertleştirilmemiş

`bin/cli/commands/redis.mjs:9` — `docker.io/redis:7-alpine`; kök Compose 8.6.5 kullanıyor. `:18` — Podman/Docker tercihi. `run` argümanlarında `--cap-drop`, `--security-opt`, `--pids-limit`, `--memory` yok. Redis 8.6.5'e hizala; loopback bind'i koru; container hardening ekle.

#### O8. WSL / OrbStack sandbox sertleştirmesi eksik

`src/lib/skills/containerProvider.ts`:

- Docker `:119-127` — `--cap-drop ALL`, `no-new-privileges`, `--pids-limit` var.
- Apple `:176-183` — aynı flag'ler var.
- WSL `:231-242` — `--cap-drop` / `no-new-privileges` yok.
- OrbStack `:284-295` — aynı eksik.
- OrbStack `:266` — `probeCommand("orbstack")`; OrbStack'ın CLI adı genellikle `orb` / `orbctl`; bu sağlayıcı büyük olasılıkla hiç algılanmıyor. Bu makinede `orbstack`/`orb`/`orbctl` yok.

WSL/OrbStack'e aynı flag'leri ekle; OrbStack binary adını doğrula; her sağlayıcı için test ekle.

#### O9. `host` profili `rw` mount + doküman “read-only” diyor

`docker-compose.yml` `host` profili `~/.codex`, `~/.claude` gibi config'leri `rw` mount ediyor. `docs/guides/DOCKER_GUIDE.md:93` bunları read-only olarak anlatıyor. Bu, doküman hatası olduğu kadar güvenlik sürprizi: compromise olan uygulama host AI CLI credential'larını değiştirebilir. Ya `ro` yap ya da dokümanda açıkça “rw, güvenilen tek kullanıcı” de.

#### O10. Bun imajı runtime parity eksikleri

- `check-permissions.sh` yok; bind mount izin sorunları sessizce fail edebilir.
- `docker/ensure-docker-base-path.mjs` runtime patch'i yok; `OMNIROUTE_BASE_PATH` runtime değişimi uygulanmaz.
- `OMNIROUTE_MEMORY_MB` var ama `NODE_OPTIONS` heap ayarı yok; Bun V8 flag'lerini yok sayabilir.
- `runner-web` apt Chromium kullanıyor; Node imajı Playwright Chromium kullanıyor; sürüm parity'si doğrulanmalı.
- Bun 1.4.3 vs proje pinli 1.4.2.

Bun imajı “best-effort” ise bunu netleştir; production iddiası varsa parity'yi tamamla.

---

### Düşük

- `src/i18n/messages/tr.json:10658` — `"Liman işçisi"`; Docker için yanlış. Tüm dillerde `Docker` korumalı terim olarak kullanılmalı.
- `docs/guides/DOCKER_GUIDE.md` birçok yerde bayat: Redis 7 / 8.6.2, `core,redis` profilleri, host read-only, Turbopack varsayılanı, qodercli eksikliği.
- `docker-compose.yml` üst profil listesinde `codex-app-server` yok; Docker Guide tablosunda `memory`, `bifrost`, `codex-app-server` yok.
- `docs/providers/ZED-DOCKER.md:27-28` yalnızca `/.dockerenv` ve cgroup v1'e bakıyor; `src/shared/utils/containerEnv.ts` Podman/containerd/kubepods gibi ek işaretler destekliyor.
- `docker-compose.selfhost.yml:38` `:latest`; `autoUpdate` için pratik ama tekrarlanabilirlik için pinlenebilir.
- VNC CDP bridge token kontrolü substring; `timingSafeEqual` değil. Token yüksek entropili olduğu için düşük, ama düzeltilmeli.
- VNC browser için healthcheck yok; s6 servisinin hazır olması `waitForCdpReady` ile telafi ediliyor.

### İyi kontroller

- Ana `Dockerfile`: `npm ci --ignore-scripts`, lockfile, non-root, `REQUIRE_API_KEY=true`, `check-permissions.sh`, healthcheck.
- `contrib/vps/compose.yaml`: `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, `mem_limit`, `cpus`, `tmpfs`, loopback, `REQUIRE_API_KEY` default true.
- `docker/devin-bridge/compose.yml`: `read_only`, `cap_drop`, `no-new-privileges`, `user: 10001`, internal network, egress guard.
- `docker/chatgpt-web-codex-browser/cdp-proxy.mjs`: token `timingSafeEqual` + fail-closed.
- `docker/vnc-browser/chromium/svc-de-run`: token yoksa bridge'i başlatmıyor; fail-closed.
- Compose host portları: Redis, Qdrant, Bifrost, CLIProxyAPI, VNC/CDP loopback.
- `docker-publish`: multi-arch, manifest, smoke, Trivy, SBOM, latest promosyonu.
- `check-permissions.sh` ve `healthcheck.mjs`: operasyonel olarak değerli.

---

## 6. Konteyner varyantı desteği

### Birinci taraf

- **Docker / Docker Compose** — ana yüzey. `Dockerfile`, dört Compose dosyası, yayın hattı, VNC, ChatGPT Web Codex, Devin bridge, CLI Redis, auto-update.
- **Podman** — `contrib/podman` Quadlet, `bin/cli/commands/redis.mjs` (Podman öncelikli), `containerProvider` Podman sağlayıcısı, `liveContainerHarness.ts`.
- **Apple Container** — `src/lib/skills/containerProvider.ts` `AppleContainerProvider`; `container` CLI; `--cap-drop ALL` + `no-new-privileges` var; test yok.
- **WSL Container** — `wslc` sağlayıcısı; `--cap-drop` / `no-new-privileges` yok; test yok.
- **OrbStack** — `containerProvider` sağlayıcısı var; `orbstack` binary adı büyük olasılıkla yanlış; flag'ler eksik; test yok.
- **BuildKit / Buildx** — CI'da `docker/setup-buildx-action` ve `build-push-action`; ürün runtime'ı değil.
- **Docker socket / Docker CLI** — `autoUpdate`, `cli` profili, VNC, sandbox, CLI Redis.

### Birinci taraf olmayan / tesadüfi

- **nerdctl** — ilk taraf entegrasyon yok.
- **Buildah** — ilk taraf entegrasyon yok; yalnızca Dockerfile yorumlarında `podman build` geçiyor.
- **containerd** — yalnızca VM rehberinde `containerd.io` kurulumu ve cgroup algılama; runtime seçeneği değil.
- **devcontainer** — `.devcontainer` yok.
- **Testcontainers** — bağımlılık yok.
- **dockerode** — bağımlılık yok; tüm Docker etkileşimi CLI üzerinden.
- **Docker Compose (composer tool)** — `open-sse/utils/composerToolCalls.ts` yanlış pozitif; Docker Compose değil.
- **`src/lib/container.ts`** — DI container; Docker değil.

Önemli nokta: Docker bağımlılığı yalnızca deployment değil; **VNC login, ChatGPT Web Codex, auto-update, CLI Redis, rollback ve test harness** gibi ürün özelliklerine de bağlı.

---

## 7. Bayat / kırık referanslar

Öne çıkan 16 madde:

1.  `docs/guides/DOCKER_GUIDE.md:187` — Redis `7-alpine`; gerçek `redis:8.6.5-alpine`.
2.  `docs/guides/DOCKER_GUIDE.md:223` — prod Redis `8.6.2`; gerçek `8.6.5`.
3.  `docs/guides/DOCKER_GUIDE.md:174` — `--profile core,redis`; `core` ve `redis` profilleri yok.
4.  `docs/guides/DOCKER_GUIDE.md:93` — host mount'ları read-only diyor; config mount'ları `rw`.
5.  `docs/guides/DOCKER_GUIDE.md:250` — builder'da Turbopack varsayılanı diyor; gerçek `OMNIROUTE_USE_TURBOPACK=0` (webpack); ayrıca `npm ci --legacy-peer-deps` yazıyor, gerçek `--ignore-scripts`.
6.  `docs/guides/DOCKER_GUIDE.md:251` — `runner-cli` listesinde `qodercli` yok; gerçek imajda var.
7.  `docs/guides/DOCKER_GUIDE.md` profil tablosu — `memory`, `bifrost`, `codex-app-server` eksik.
8.  `docker-compose.prod.yml:14-26` — başlık `runner-base` diyor; `:53` `runner-cli`.
9.  `contrib/podman/omniroute-redis.container:7` — Redis 7; ana stack 8.6.5.
10. `README.md:1123-1124` — `Dockerfile.bun` “production” diyor; yayınlanan `-bun` etiketleri ve keyless duruşu dokümante değil.
11. `docs/providers/ZED-DOCKER.md:27-28` — yalnızca `/.dockerenv` + cgroup v1; kod daha fazla işaret destekliyor.
12. `docs/guides/LOCAL_OLLAMA_DOCKER.md` — `docker exec` ve `0.0.0.0` bind; auth/REQUIRE_API_KEY rehberi yok.
13. `.github/workflows/deploy-vps.yml:4-6` — Docker publish ile tetikleniyor; `:60` npm kuruyor. Gizli coupling.
14. `fly.toml:9` — boş `[build]`; Fly `Dockerfile` kullanıyor. Dockerfile kaldırılırsa Fly deploy kırılır.
15. `docs/ops/VM_DEPLOYMENT_GUIDE.md` — Docker/containerd kuruyor; Docker-free politika ile çelişir.
16. `src/i18n/messages/tr.json:10658` — `"Liman işçisi"`; diğer dillerde de transliterasyon/çeviri sorunları.

Ek olarak:

- `bin/rollback.sh:42-44,91-97` — Docker rollback yolu `docker-compose.prod.yml` ve yerel `omniroute:<version>` imajına bağlı.
- `bin/cli/commands/redis.mjs` — Redis 7 ve sertleştirme eksikleri.
- `tests/integration/liveContainerHarness.ts` — `/home/markus/...` gibi operatöre özel varsayılanlar; CI'da çalışmaz.

---

## 8. Politika çelişkisi

Global `AGENTS.md`:

“Docker-Free system. Never use Docker (or variants) on this system. If any project offers Docker, remove all Docker in the project.”
Proje `AGENTS.md` ise Docker'ı açıkça destekliyor:

- Node tek desteklenen runtime; Bun best-effort.
- Docker imajları, Compose, Fly, VPS, Podman, VNC, Devin bridge dokümante.
- `Dockerfile` ve `docker-publish.yml` aktif bakımda.

Bu nedenle iki farklı politika var:

1. **Çalışma istasyonu politikası** — bu makinede Docker/Podman çalıştırma.
2. **Ürün politikası** — yayınlanan konteyner artefaktları ve konteyner tabanlı özellikler.

Docker'ı literal olarak “projeden tamamen kaldır” denirse şunlar etkilenir:

- Yayınlanan Docker Hub/GHCR imajları ve `docker-publish.yml`.
- Fly.io deploy (`fly.toml` → `Dockerfile`).
- `contrib/vps`, `contrib/podman`, Compose stack'leri.
- VNC browser login.
- ChatGPT Web Codex sidecar.
- Devin bridge test harness.
- CLI Redis.
- Auto-update Docker modları.
- `bin/rollback.sh --method docker`.
- 159 test dosyasındaki referansların bir kısmı.
- Docker rehberleri ve i18n çevirileri.

Ancak `deploy-vps.yml` zaten npm ile kuruyor; yani VPS deploy'u Docker'sız çalışabilir. Fly için buildpack veya ayrı bir Dockerfile gerekir. VNC/ChatGPT Web Codex için Podman/Apple Container alternatifi veya özellik kaldırma gerekir.

**Öneri:** Önce kapsamı ayır:

- “Bu workstation'da Docker çalıştırma” → CI dışı, dokümantasyon ve scriptlerde netleştir.
- “Ürün Docker artefaktlarını kaldır” → ayrı, planlı bir migration; yayınlanan imajları ve kullanıcıları etkiler.
- “Sadece Docker socket/privileged kullanımını kaldır” → en güvenli ve en düşük riskli adım.

Docker artefaktlarını, yerine geçen çözümler hazırlanmadan silmek önerilmez.

---

## 9. CI / test etkisi

- `docker-publish.yml` release için kritik; kaldırılırsa Docker Hub/GHCR yayını, Fly için imaj, VPS tetikleyicisi ve smoke/Trivy/SBOM kaybolur.
- `ci.yml:227` hadolint `docker run` kullanıyor; Docker-free workstation'da static hadolint binary'ye geçilebilir.
- `semgrep.yml:18` GitHub Actions job container; yerel Docker değil, ama konteyner runtime'ı kullanır.
- `tests/security/test-docker-hardening.sh` ve `test-cli-runtime.sh` gerçek Docker build/run yapıyor; CI'da değil. Docker kaldırılırsa bu testler manuel/dead kalır.
- `tests/integration/liveContainerHarness.ts` Podman kullanıyor; `RUN_LIVE_WIRE_CAPTURE=1` ile gated; CI'da değil.
- `tests/unit/issue-13679...` yalnızca `Dockerfile` + `fly.toml`; `Dockerfile.bun` kapsam dışı.
- `tests/unit/vnc-session-docker-args.test.ts` yalnızca ağ/token; VNC hardening kapsam dışı.
- `tests/unit/skills-builtins-sandbox.test.ts` yalnızca Docker argümanları; Apple/WSL/OrbStack kapsam dışı.
- `compose-*` testleri YAML/metin tabanlı; runtime Compose doğrulaması değil.

Test boşlukları:

- Bun imajı için `REQUIRE_API_KEY` testi yok.
- VNC için `--cap-drop` / `no-new-privileges` / VNC password testi yok.
- `-web` / `-bun` imajları için smoke/scan yok.
- WSL/OrbStack sandbox argümanları için test yok.
- `docker-compose.prod.yml` CDP network ve `runner-base` hedefi için test yok.
- Duplicate `container_name` için `docker compose config` testi yok.

---

## 10. Öncelikli aksiyonlar

### P0 — güvenlik / yayın bütünlüğü

1. **`Dockerfile.bun`'a `ENV REQUIRE_API_KEY=true` ekle** ve `issue-13679` testini iki Dockerfile için parametreleştir. Bun imajı “best-effort” ise `-bun` etiketlerini yayınlama veya açıkça işaretle.
2. **`cli` profilinden `docker.sock` mount'unu kaldır**; `DOCKER_UPDATE_TRIGGER_FILE` / `docker-trigger` moduna geç. Repo mount'unu `ro` yap; socket gerekliyse ayrı, açıkça opt-in override'a taşı.
3. **VNC browser'ı sertleştir:** `CUSTOM_USER`/`PASSWORD` veya per-session token, `--cap-drop ALL`, `--security-opt no-new-privileges`, `--pids-limit`, `--memory`, digest-pinli `linuxserver/chromium`, healthcheck.
4. **Bun build'i düzelt:** `bun.lock` ekle, `.dockerignore`'dan çıkar, `--frozen-lockfile --ignore-scripts`, `OMNIROUTE_BUILD_MEMORY_MB` ARG'si, Bun sürüm parity'si.
5. **CI'yi tüm yayınlanan varyantlara genişlet:** `-web`, `-bun`, `-web-bun` için smoke + Trivy + SBOM.

### P1 — yapılandırma / mimari

6.  Compose'da `REQUIRE_API_KEY` varsayılanını `true` yap; `.env.example` container bölümünü hizala; sidecar ağlarını segmentlere ayır.
7.  `docker-compose.prod.yml` hedefini `runner-base` yap; başlığı düzelt.
8.  Ana Compose stack'ine `cap_drop`, `security_opt`, `pids_limit`, `mem_limit` ekle; `contrib/vps`'i template al.
9.  Prod'a `chatgpt-web-codex-net` izolasyonunu ekle.
10. `API_HOST` / `LIVE_WS_HOST` ile publish edilen portları düzelt: ya `0.0.0.0` bind + loopback publish, ya da portları kaldır.
11. Podman Quadlet'i `127.0.0.1:...` bind + `DropCapability=ALL` + `NoNewPrivileges=true` + limitlerle sertleştir.
12. `docker-compose.ollama-local.yml`'u dev-only olarak işaretle; loopback + pinli sürüm + `env_file` ekle.
13. CLI Redis'i 8.6.5'e hizala; `--cap-drop ALL`, `--security-opt no-new-privileges`, limitler ekle.
14. Tüm action'ları SHA'ya, taban imajları digest'e pinle; `linuxserver/chromium:latest`'i kaldır.
15. WSL/OrbStack sandbox'larına eksik flag'leri ekle; OrbStack binary adını doğrula; test yaz.

### P2 — doküman / test / operasyon

16. `DOCKER_GUIDE`'daki Redis, profil, Turbopack, qodercli, host read-only hatalarını düzelt.
17. `docker-compose.prod.yml` başlığını ve `Dockerfile.bun` dokümantasyonunu hizala.
18. `tr.json` ve diğer dillerde `deployDockerTitle` için `Docker` kullan; glossary kontrolünü genişlet.
19. `ZED-DOCKER.md` algılama bölümünü `containerEnv.ts` ile hizala.
20. `fly.toml` / `bin/rollback.sh` / `deploy-vps.yml` Docker bağımlılığını belgele; Docker'sız alternatifleri tanımla.
21. VNC/CDP token karşılaştırmasını `timingSafeEqual`'a çevir; VNC healthcheck ekle.
22. `liveContainerHarness.ts` içindeki `/home/markus/...` varsayılanlarını kaldır; env ile yapılandırılabilir yap.
23. Gerçek Docker/Podman integration testlerini CI'a uygun hale getir veya açıkça “manual security test” olarak işaretle.

---

## 11. Doğrulama sınırları

Bu rapor **statik** analizdir. Aşağıdakiler çalıştırılmadı:

- `docker build`, `docker run`, `docker compose config/up`.
- Podman build/run.
- Trivy, hadolint, SBOM üretimi.
- Fly deploy / `fly config validate`.
- VNC/CDP runtime testi.
- `API_HOST` / `LIVE_WS_HOST` port erişilebilirlik testi.
- Duplicate `container_name` için Compose doğrulaması.

Runtime doğrulaması gereken başlıklar:

- `API_HOST=127.0.0.1` + published 20129 gerçekten erişilemiyor mu?
- `LIVE_WS_HOST=127.0.0.1` + published 20132 aynı şekilde.
- VNC noVNC UI gerçekten parolasız mı?
- `linuxserver/chromium:latest` sürümünde `CUSTOM_USER`/`PASSWORD` varsayılanı ne?
- `docker compose --profile base --profile web` duplicate `container_name` nedeniyle hata veriyor mu?
- Bun imajı `bun dev/run-standalone.mjs` ile `NODE_OPTIONS` heap flag'lerini kabul ediyor mu?

---

## 12. Sonuç

OmniRoute'un Docker desteği olgun ve geniş: ana Node imajı iyi sertleştirilmiş, VPS/Devin örnekleri güçlü, yayın hattı ciddi. Ancak üç yapısal sorun öne çıkıyor:

1. **Bun imajı ikinci sınıf güvenlik duruşunda** — yayınlanıyor ama Node imajının `REQUIRE_API_KEY`, lockfile, `--ignore-scripts` ve CI kapsamı kontrollerini taşımıyor.
2. **Opt-in yüzeyler host root'a çok yakın** — `cli` profili Docker socket + repo `rw`; `host` profili AI CLI config'lerini `rw`; VNC browser parolasız olabilir.
3. **Compose ağ modeli düz** — loopback host bind'i sibling container'lara karşı korumuyor; keyless `/v1` ve credential'lı sidecar'lar aynı ağda.

Docker'ı “tamamen kaldırma” kararı, yayınlanan imajları, Fly/VPS/Podman dağıtımlarını, VNC/ChatGPT Web Codex özelliklerini ve testleri etkiler. Bu nedenle önce kapsam ayrımı yapılmalı; P0 maddeleri ise Docker kaldırılsa da kaldırılmasa da düzeltilmesi gereken gerçek güvenlik açıkları.
İsterseniz P0 maddelerini ayrı bir worktree'de, mevcut 9 değişikliğe dokunmadan yama olarak hazırlayabilirim.
