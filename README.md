# VPN Torrent Store (Umbrel)

Community App Store com **qBittorrent** e **Transmission** atrás do **Gluetun**, com kill switch.

## Como o kill switch funciona (2 camadas)
1. `network_mode: "service:gluetun"` — o cliente torrent **não tem rede própria**; usa só a stack do Gluetun. Se o Gluetun parar, o cliente fica sem rede.
2. Firewall do Gluetun (`FIREWALL=on`) — bloqueia qualquer tráfego que não saia pelo túnel VPN. Se o túnel cair, nada sai. Só são permitidas as redes locais (`FIREWALL_OUTBOUND_SUBNETS`) para o WebUI funcionar.

## Instalação

### 1. Publicar a store
1. No GitHub: **Use this template** em `getumbrel/umbrel-community-app-store`, ou cria um repo novo e copia estes ficheiros.
2. Substitui `CHANGE-ME` nos `umbrel-app.yml` pelo teu utilizador GitHub.
3. No Umbrel: **App Store → ⋯ → Community App Stores → Add** e cola o URL do repo.

### 2. Configurar a VPN (ANTES de instalar a app)
Por SSH no Umbrel:
```bash
mkdir -p ~/umbrel/app-data/vpn-qbittorrent
nano ~/umbrel/app-data/vpn-qbittorrent/gluetun.env   # usa examples/gluetun.env.example
chmod 600 ~/umbrel/app-data/vpn-qbittorrent/gluetun.env
```
Repete para `vpn-transmission` se instalares as duas.
> As credenciais ficam fora do repo, por isso não são apagadas em updates nem publicadas.

### 3. Instalar e verificar
```bash
# logs do Gluetun — deve mostrar "Public IP address is x.x.x.x (Portugal ...)"
docker logs vpn-qbittorrent_gluetun_1 --tail 30

# IP visto de dentro do contentor torrent (tem de ser o da VPN)
docker exec vpn-qbittorrent_gluetun_1 wget -qO- https://ipinfo.io

# password temporária do qBittorrent (utilizador: admin)
docker logs vpn-qbittorrent_server_1 2>&1 | grep -i password
```

### 4. Testar o kill switch
```bash
docker stop vpn-qbittorrent_gluetun_1          # o qbittorrent fica sem rede
# ou bloquear a VPN sem parar o contentor:
docker exec vpn-qbittorrent_gluetun_1 wget -T5 -qO- https://ipinfo.io   # deve falhar se o túnel estiver em baixo
```
Também podes usar um torrent de teste de IP (ex.: ipleak.net) — só deve aparecer o IP da VPN.

## Notas
- **Estrutura (igual às apps oficiais)**: `${APP_DATA_DIR}/data/config` (config do cliente), `${APP_DATA_DIR}/data/gluetun` (estado do Gluetun) e `gluetun.env` na raiz da pasta da app. O serviço do cliente chama-se `server`, como no `transmission` oficial.
- **Portas de peers**: as apps oficiais publicam `51413`; aqui **não**, porque o tráfego tem de sair só pela VPN. Usa port forwarding do fornecedor (`VPN_PORT_FORWARDING=on`) ou `FIREWALL_VPN_INPUT_PORTS`, e define a mesma porta no cliente.
- **Widgets**: só o Transmission os inclui (o `widget-server` oficial aponta para o contentor do Gluetun).
- **Permissão** `STORAGE_DOWNLOADS` + `storage.dataRoot: data` seguem o manifesto oficial (umbrelOS 2.0).
- **Pastas**: downloads em `~/umbrel/data/storage/downloads` (pasta "Downloads" do Umbrel).
- **Nome dos contentores**: `APP_HOST` = `<app-id>_<serviço>_1`. Se mudares o ID da store/app, atualiza-o no `docker-compose.yml`.
- **Duas apps ao mesmo tempo** usam duas ligações VPN em simultâneo; verifica o limite de dispositivos do teu fornecedor.
- **Versões**: `latest` é conveniente, mas fixa versões (ou digests) para updates previsíveis.
- **Sub-redes**: `10.21.0.0/16` é a rede docker do Umbrel; ajusta `192.168.0.0/16` à tua LAN (ou remove se não precisares de aceder a nada local).
- Convenções seguidas das apps oficiais (`getumbrel/umbrel-apps`): sem campo `version`, `restart: on-failure`, `app_proxy` com `APP_HOST: <app-id>_<serviço>_1`, dados em `${APP_DATA_DIR}/data/...`. Nas apps oficiais as imagens levam tag + `@sha256:` — faz o mesmo aqui antes de publicar (`docker pull` + `docker inspect --format="{{index .RepoDigests 0}}"`).
- Requer um Docker Compose recente (o `env_file` com `required: false` precisa de Compose ≥ 2.24). Se o teu Umbrel for mais antigo, remove `required: false` e cria o `gluetun.env` antes de instalar.
