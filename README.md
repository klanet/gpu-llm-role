# Автоматизация развёртывания GPU-сервера для LLM (Ansible)

**Задача:** Написать Ansible Playbook для настройки GPU-сервера (Ubuntu + NVIDIA) и запуска LLM-модели с доступом через API.

---

## Что реализовано

| Требование задачи | Реализация |
|-------------------|------------|
| **ОС — Ubuntu, GPU — NVIDIA** | Проверка Ubuntu 22.04+, наличия GPU на PCI шине |
| **Настроить драйверы** | Роль `nvidia_driver`: blacklist nouveau, `nvidia-driver-<N>-server`, persistence-демон, авто-reboot при необходимости |
| **Настроить Docker и окружение** | Роль `docker_nvidia`: Docker CE + Compose plugin, NVIDIA Container Toolkit, тест `docker run --gpus all nvidia-smi` |
| **Развернуть AI-модель (Ollama / vLLM)** | Роль `llm_stack`: docker-compose с выбором движка (`llm_engine: vllm \| ollama`), моделью (`llm_model`), автозагрузкой весов |
| **Автоматический запуск** | Docker в systemd, контейнеры `restart: unless-stopped` → стек поднимается после перезагрузки |
| **Проверка работоспособности** | Smoke-тест в плейбуке: драйвер, GPU в контейнере, загрузка модели, реальный chat-запрос, 401 без ключа |
| **README с инструкцией и примером API** | Этот файл |
| **Мониторинг GPU (доп.)** | `dcgm-exporter` → метрики на `:9400/metrics` (Prometheus/Grafana) |
| **Защита API (доп.)** | nginx gateway: TLS, Bearer-ключи, IP allow-list, rate limit, порт модели не публикуется |

---

## Архитектура

```
Клиент ──HTTPS + Bearer-ключ──► nginx (gateway) ──► vLLM / Ollama ──► GPU
                                 │ TLS, API-ключи,       (порт не публикуется,
                                 │ rate limit, allow-list  доступен только шлюзу)
                                 ▼
                           dcgm-exporter :9400 ──► Prometheus/Grafana
systemd timer (2 мин) ──► проверка API + nvidia-smi ──► рестарт при 3 сбоях
```

---

## Быстрый старт

```bash
# 1. Зависимости
ansible-galaxy collection install -r requirements.yml

# 2. Инвентарь — укажите ваш сервер
$EDITOR inventory/hosts.yml

# 3. Секреты (API-ключи, HF-токен для gated-моделей)
cp group_vars/gpu_servers/vault.example.yml group_vars/gpu_servers/vault.yml
openssl rand -hex 32                              # сгенерируйте ключ
# Впишите в vault.yml:  llm_api_keys: ["sk-<hex>"]
ansible-vault encrypt group_vars/gpu_servers/vault.yml

# 4. Переменные — выберите движок и модель
$EDITOR group_vars/gpu_servers/vars.yml
# llm_engine: vllm        # или ollama
# llm_model: Qwen/Qwen2.5-7B-Instruct-AWQ   # HF-id для vLLM, тег для Ollama

# 5. Запуск (первый раз: скачивание весов 10+ мин)
ansible-playbook site.yml --ask-vault-pass
```

### Селективный запуск

```bash
ansible-playbook site.yml --tags nvidia   # только драйвер
ansible-playbook site.yml --tags docker   # только Docker
ansible-playbook site.yml --tags llm      # только LLM стек
ansible-playbook site.yml --check --diff  # dry-run
```

---

## Структура проекта

```
.
├── site.yml                          # главный плейбук (4 роли)
├── inventory/hosts.yml               # серверы (отредактируйте IP/ключ)
├── group_vars/gpu_servers/
│   ├── vars.yml                      # все настройки
│   └── vault.example.yml             # пример секретов
├── requirements.yml                  # community.general, community.docker
├── ansible.cfg                       # конфиг Ansible
└── roles/
    ├── common/                       # базовая ОС, UFW, проверки
    ├── nvidia_driver/                # драйвер NVIDIA
    ├── docker_nvidia/                # Docker + NVIDIA Toolkit
    └── llm_stack/                    # LLM + nginx + dcgm + watchdog
```

---

## Основные переменные (`group_vars/gpu_servers/vars.yml`)

```yaml
# ───────── NVIDIA ─────────
nvidia_driver_package: nvidia-driver-570-server  # должен поддерживать CUDA образа
nvidia_allow_reboot: true                        # ребут если драйвер не загрузился

# ───────── LLM ─────────
llm_engine: vllm                                 # vllm | ollama
llm_model: Qwen/Qwen2.5-7B-Instruct-AWQ          # HF-id (vLLM) или тег (Ollama)
vllm_image: vllm/vllm-openai:v0.10.2             # ЗАФИКСИРУЙТЕ версию
vllm_extra_args:                                 # под VRAM
  - "--dtype"; "half"
  - "--max-model-len"; "8192"
  - "--gpu-memory-utilization"; "0.90"

# ───────── API-шлюз (nginx) ─────────
api_tls_enabled: true                            # самоподписанный, в проде — настоящий
api_port: 443
api_rate_limit: 10r/s
api_allowed_cidrs: ["0.0.0.0/0"]                 # ОБЯЗАТЕЛЬНО ограничьте подсетями клиентов

# ───────── Мониторинг GPU ─────────
dcgm_enabled: true
dcgm_bind_address: 127.0.0.1                     # для Prometheus — приватный IP
```

**Секреты** в `vault.yml` (ansible-vault):
```yaml
llm_api_keys:
  - "sk-ваш-ключ"
hf_token: ""  # для gated-моделей (Llama и др.)
```

---

## Примеры API-запросов

### cURL
```bash
curl -k https://<SERVER_IP>/v1/chat/completions \
  -H "Authorization: Bearer sk-<ваш_ключ>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-7B-Instruct-AWQ",
    "messages": [{"role": "user", "content": "Объясни, что такое GPU, в двух предложениях."}],
    "max_tokens": 200
  }'
```
- **Стриминг:** добавьте `"stream": true`
- **Список моделей:** `GET /v1/models`
- `-k` нужен для самоподписанного сертификата (в проде замените на настоящий)

### Python (OpenAI-совместимый клиент)
```python
from openai import OpenAI
import httpx

client = OpenAI(
    base_url="https://<SERVER_IP>/v1",
    api_key="sk-<ваш_ключ>",
    http_client=httpx.Client(verify=False)  # только для самоподписанного cert
)

r = client.chat.completions.create(
    model="Qwen/Qwen2.5-7B-Instruct-AWQ",
    messages=[{"role": "user", "content": "Привет!"}],
)
print(r.choices[0].message.content)
```

---

## Проверка работоспособности

Плейбук выполняет автоматические проверки:
- `nvidia-smi` — драйвер работает
- `docker run --gpus all nvidia-smi` — GPU доступен в контейнере
- Модель загружена и отвечает на `/v1/models`
- Реальный chat-completion запрос проходит
- Без ключа возвращается `401`

### Вручную на сервере
```bash
nvidia-smi                                    # драйвер
docker compose -f /opt/llm/docker-compose.yml ps  # контейнеры
docker logs -f llm                            # логи модели (загрузка весов)
curl -k https://localhost/healthz             # живость шлюза
systemctl list-timers llm-healthcheck.timer   # watchdog таймер
journalctl -t llm-healthcheck                 # события автовосстановления
```

---

## Мониторинг GPU (Prometheus + Grafana)

`dcgm-exporter` на `:9400/metrics` — утилизация, VRAM, температура, мощность, XID-ошибки.

**Prometheus:**
```yaml
- job_name: gpu
  static_configs:
    - targets: ["<PRIVATE_IP>:9400"]
```

**Grafana:** дашборд **NVIDIA DCGM Exporter** (ID 12239).  
Алерты: температура > 85°C, VRAM > 95%, XID-ошибки, `up == 0`.

---

## Безопасность: что из коробки

- ✅ Порт модели (8000/11434) **не опубликован** наружу
- ✅ TLS на шлюзе (самоподписанный, замените в проде)
- ✅ Bearer-авторизация по списку ключей
- ✅ IP allow-list (`api_allowed_cidrs`) — **ограничьте!**
- ✅ Rate limit 10 r/s (burst 20)
- ✅ Ключи в `ansible-vault`, конфиги `0600`
- ✅ UFW deny-by-default (только SSH + API порт)
- ✅ Драйвер и Docker **не обновляются автоматически** (apt blacklist)

> Docker обходит UFW для опубликованных портов — защита на уровне nginx (`api_allowed_cidrs`) и привязка dcgm к конкретному IP.

---

## Дополнительные улучшения (roadmap)

- **Настоящий TLS:** Let's Encrypt / Caddy / Cloudflare Tunnel; mTLS
- **Управление ключами:** LiteLLM Proxy (ключи на пользователя, бюджеты, fallback)
- **Секреты:** HashiCorp Vault / SOPS вместо ansible-vault
- **Масштабирование:** несколько GPU-нод, `--tensor-parallel-size`, k3s + GPU Operator, KServe
- **Кэш образов/весов:** приватный registry + S3/NFS для независимости от Hugging Face
- **Логи/алерты:** Loki/Promtail, Alertmanager → Telegram/Slack, node-exporter
- **Харденинг хоста:** fail2ban, SSH только по ключам из VPN, unattended-upgrades (не-GPU)
- **CI:** ansible-lint + Molecule, тестовый прогон на GPU-ноде

---

## Известные нюансы

| Ситуация | Решение |
|----------|---------|
| Старые GPU (T4, V100, Turing) | Нет bf16 → `--dtype half`; для vLLM ставить `vllm_env: { VLLM_USE_V1: "0" }` или использовать Ollama |
| Мало VRAM (16 ГБ) | 7B fp16 ≈ 15 ГБ → берите AWQ/GPTQ кванты, уменьшайте `--max-model-len` |
| Gated-модели (Llama) | Добавьте `hf_token` в vault |
| Ollama: неверный формат модели | Указывайте тег (`qwen2.5:7b-instruct`), а не HF ID (`Qwen/...`) |
| Смена драйвера | При `nvidia_allow_reboot: true` сервер перезагрузится — планируйте окно |
| Версии образов | Закреплены в vars — проверяйте актуальные теги перед запуском |

---

## Лицензия

MIT — используйте, меняйте, делитесь.