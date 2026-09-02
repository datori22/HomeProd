# HomeProd — Servidor Doméstico Self-Hosted

Infraestrutura pessoal construída do zero para substituir serviços pagos de nuvem (backup, streaming de mídia, gestão financeira) por soluções self-hosted, com foco em segurança, automação e observabilidade — e como projeto prático de estudo para a trilha de Infraestrutura/SRE.

## Motivação

O objetivo não era só "rodar uns containers": era simular, em escala doméstica, as mesmas preocupações de um ambiente de produção real — hardening de acesso, rede segmentada por confiança, backup com retenção, monitoramento com alertas, e documentação de cada decisão de arquitetura (inclusive as que foram revertidas).

## Stack

| Camada | Tecnologia |
|---|---|
| SO | Debian 13 (mínimo, sem ambiente desktop) |
| Virtualização de serviços | Docker + Docker Compose |
| Rede privada / acesso remoto | Tailscale (WireGuard-based mesh VPN) |
| Firewall | UFW, com regras por origem (VPN vs. rede local) |
| Proteção contra brute-force | Fail2ban |
| Atualizações | unattended-upgrades |
| DNS local / bloqueio de rastreadores | Pi-hole v6 |
| Fotos e vídeos | Immich (alternativa ao Google Fotos) |
| Streaming de mídia | Jellyfin |
| Backup em nuvem | rclone (sync incremental com retenção de deletados) |
| Controle financeiro | Firefly III |
| Media center / cliente | Kodi (DLNA) |
| Alertas / automação | Scripts Bash + cron + Telegram Bot API |

## Arquitetura de rede

Modelo de confiança em duas camadas:

- **Tailscale (VPN mesh)**: acesso administrativo (SSH) e a serviços específicos de qualquer lugar do mundo, autenticado por dispositivo — sem porta nenhuma exposta à internet pública.
- **Rede local (LAN)**: liberada como camada de confiança adicional para uso doméstico direto (ex.: descoberta DLNA do Kodi, que depende de multicast e não atravessa VPN da mesma forma).

```
Internet
   │
   ├── Tailscale (WireGuard) ──► SSH, painéis de serviço (acesso remoto)
   │
Rede doméstica (LAN)
   │
   └── homesrv01 (Debian 13)
        ├── Docker
        │     ├── Immich (fotos/vídeos)
        │     ├── Jellyfin (mídia)
        │     ├── Firefly III (financeiro)
        │     └── Pi-hole (DNS local)
        ├── UFW (deny por padrão, allow por origem)
        ├── Fail2ban
        └── cron
              ├── rclone sync (backup noturno)
              └── health check + backup check (alertas Telegram)
```

## Segurança implementada

- **SSH**: autenticação apenas por chave pública (`PasswordAuthentication no`), validado via `sshd -T` (não só lido do arquivo de config — evita falso positivo de diretiva comentada assumindo valor padrão).
- **UFW**: política padrão `deny incoming`, liberação explícita por origem (faixa da VPN vs. faixa da LAN) e por porta/serviço, cada regra documentada com `comment`.
- **Fail2ban**: bloqueio automático de tentativas de força bruta.
- **Atualizações automáticas** de segurança via `unattended-upgrades`.
- **Segredos fora do versionamento**: credenciais de API (bot do Telegram, etc.) isoladas em arquivo de config separado, nunca commitado.

## Monitoramento e automação

Dois scripts Bash rodando via cron, com alerta via Telegram (silencioso quando está tudo OK — só notifica problema real):

- **Health check diário**: containers parados/unhealthy, uso de disco, uso de memória (lido via `/proc/meminfo` para maior portabilidade entre sistemas).
- **Verificação de backup**: confirma que o job noturno de sincronização rodou e não houve erro no dia.

## Backup

Sync incremental de nuvem pessoal via `rclone`, com uma decisão de design específica: em vez de espelhamento puro (que apaga local ao apagar na nuvem), o backup preserva arquivos removidos da origem em um diretório de arquivo morto (`--backup-dir`), permitindo reduzir uso de armazenamento na nuvem sem perder histórico localmente.

## Problemas reais encontrados e resolvidos

Documentando aqui porque troubleshooting é a parte que mais ensina:

1. **Autenticação por senha "desligada" continuava ativa** — a diretiva `PasswordAuthentication` estava comentada no `sshd_config`, o que faz o SSH assumir o valor padrão (`yes`), não o valor pretendido. Só foi pego validando com `sshd -T` em vez de confiar na leitura visual do arquivo.
2. **Descoberta DLNA parou de funcionar após hardening do firewall** — o UFW com política restritiva bloqueava o tráfego multicast (SSDP, porta 1900/UDP) necessário para descoberta de dispositivos na rede local, mesmo com a VPN liberada. Resolvido liberando a faixa da rede local como camada de confiança adicional.
3. **Variável de ambiente descontinuada silenciosamente** — o Pi-hole v6 trocou `WEBPASSWORD` (v5) por `FTLCONF_webserver_api_password` sem erro visível: a senha antiga simplesmente era ignorada.
4. **DNS não respondia pela rede mesmo com container saudável** — o Pi-hole, por padrão, recusa consultas que chegam via porta publicada pelo Docker por considerá-las "não locais" (proteção anti-DNS-amplification). Identificado via log do container, corrigido com `FTLCONF_dns_listeningMode=all`.

## Decisões de arquitetura revertidas (e por quê)

Nem toda escolha inicial sobreviveu ao teste prático — documentando os *pivots* também:

- **Syncthing → descartado**: redundante com a solução de backup via rclone já implementada.
- **Actual Budget → revertido para Firefly III**: apesar do Actual Budget ter orçamento genuinamente compartilhado entre usuários, o Firefly III venceu por maturidade e recursos de categorização mais completos — o compartilhamento de dados entre contas foi resolvido usando um único login compartilhado, uma limitação aceita conscientemente.
- **Hetzner Storage Box → removido do escopo**: custo recorrente não justificado frente ao volume de dados atual.

## Próximos passos

- HTTPS nativo para os serviços internos via certificados do Tailscale (`tailscale serve`), eliminando acesso HTTP puro mesmo dentro da rede confiável.
- Expansão de armazenamento (upgrade de SSD/RAM).

---

*Este repositório documenta a arquitetura e as decisões técnicas do projeto. Arquivos de configuração com credenciais, IPs reais e identificadores pessoais não são versionados.*
