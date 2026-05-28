# Stardesk server em VPS/AWS com Docker

Este guia mostra como subir o servidor OSS compativel com Stardesk/RustDesk em
uma VPS, incluindo AWS EC2, usando Docker. O servidor tem dois componentes:

- `hbbs`: servidor de ID/rendezvous.
- `hbbr`: servidor de relay.

Depois disso, os clientes Stardesk precisam apontar para o mesmo endereco do
servidor.

## Quando usar

Use este caminho para testar conexao remota real com outra pessoa fora da sua
rede. O `flutter run -d chrome` ou um tunnel HTTP serve para testar UI, mas nao
substitui o servidor de rendezvous/relay necessario para conexoes remotas.

## Portas

Abra estas portas no firewall da VPS e no Security Group da AWS:

```text
TCP 21115
TCP 21116
UDP 21116
TCP 21117
TCP 21118
TCP 21119
```

Para um teste minimo, as portas principais sao `21115/tcp`, `21116/tcp`,
`21116/udp` e `21117/tcp`. Abrir `21118/tcp` e `21119/tcp` evita surpresa em
clientes e recursos auxiliares.

## AWS EC2

1. Crie uma instancia EC2 Ubuntu LTS.
2. Associe um Elastic IP se quiser manter o IP fixo.
3. No Security Group, adicione regras de entrada:

```text
Type        Protocol  Port range  Source
Custom TCP  TCP       21115       0.0.0.0/0
Custom TCP  TCP       21116       0.0.0.0/0
Custom UDP  UDP       21116       0.0.0.0/0
Custom TCP  TCP       21117       0.0.0.0/0
Custom TCP  TCP       21118       0.0.0.0/0
Custom TCP  TCP       21119       0.0.0.0/0
SSH         TCP       22          seu-ip/32
```

4. Opcionalmente crie um DNS, por exemplo `rd.seudominio.com`, apontando para o
   Elastic IP.

## Instalar Docker

Na VPS:

```sh
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $VERSION_CODENAME stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Subir servidor com o compose do repo

Copie os arquivos do repo para a VPS, ou clone o projeto na VPS. Depois crie o
arquivo de ambiente:

```sh
cd stardesk
cp .env.server.example .env.server
```

Edite `.env.server`:

```text
STARDESK_SERVER_HOST=SEU_DOMINIO_OU_IP
STARDESK_SERVER_IMAGE=ghcr.io/matheus-matta/stardesk-server:latest
```

Essa imagem e publicada pela Action do repositorio
`Matheus-Matta/stardesk`. Se quiser voltar para a imagem oficial upstream, use:

```text
STARDESK_SERVER_IMAGE=rustdesk/rustdesk-server:latest
```

Suba:

```sh
docker compose --env-file .env.server -f docker-compose.server.yml up -d
docker compose --env-file .env.server -f docker-compose.server.yml ps
```

Veja os logs:

```sh
docker logs -f stardesk-hbbs
docker logs -f stardesk-hbbr
```

Os dados persistentes ficam em `./server-data`.

## Subir servidor manualmente

Se preferir nao clonar o repo na VPS, crie a pasta:

```sh
sudo mkdir -p /opt/stardesk-server/server-data
cd /opt/stardesk-server
```

Crie `docker-compose.yml`:

```yaml
services:
  hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: stardesk-hbbs
    command: hbbs -r SEU_DOMINIO_OU_IP:21117
    volumes:
      - ./server-data:/root
    network_mode: host
    restart: unless-stopped

  hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: stardesk-hbbr
    command: hbbr
    volumes:
      - ./server-data:/root
    network_mode: host
    restart: unless-stopped
```

Troque `SEU_DOMINIO_OU_IP` pelo Elastic IP ou DNS da VPS.

Suba:

```sh
sudo docker compose up -d
sudo docker compose ps
```

Veja os logs:

```sh
sudo docker logs -f stardesk-hbbs
sudo docker logs -f stardesk-hbbr
```

## Chave do servidor

Na primeira inicializacao, o servidor gera chaves em `server-data`.
Veja a chave publica:

```sh
sudo cat /opt/stardesk-server/server-data/id_ed25519.pub
```

Guarde esse valor. Ele pode ser usado no campo `Key` do cliente Stardesk para
garantir que os clientes estao falando com o seu servidor.

## Configurar o Stardesk

No app Stardesk, abra as configuracoes de servidor/rede e preencha:

```text
ID Server: SEU_DOMINIO_OU_IP
Relay Server: SEU_DOMINIO_OU_IP
API Server: deixe vazio no servidor OSS
Key: conteudo de server-data/id_ed25519.pub
```

Faca isso nos dois clientes que vao se conectar.

Para testar:

1. Abra o Stardesk em uma maquina.
2. Copie o ID e a senha temporaria.
3. Abra o Stardesk na outra maquina.
4. Digite o ID remoto e conecte.

## Usar o app em dev

No Windows, depois de buildar:

```powershell
cd flutter
.\build\windows\x64\runner\Debug\stardesk.exe --noinstall
```

Para enviar para um amigo, compacte a pasta:

```text
flutter/build/windows/x64/runner/Debug/
```

O amigo deve abrir `stardesk.exe --noinstall` e configurar o mesmo servidor.

## Atualizar

```sh
cd /opt/stardesk-server
sudo docker compose pull
sudo docker compose up -d
```

## Parar

```sh
cd /opt/stardesk-server
sudo docker compose down
```

## Diagnostico rapido

Verifique containers:

```sh
sudo docker compose ps
```

Verifique portas ouvindo:

```sh
sudo ss -lntup | grep 211
```

Teste do seu computador:

```powershell
Test-NetConnection SEU_DOMINIO_OU_IP -Port 21115
Test-NetConnection SEU_DOMINIO_OU_IP -Port 21116
Test-NetConnection SEU_DOMINIO_OU_IP -Port 21117
```

Para UDP `21116`, valide pelo log do servidor e pelo teste real de conexao,
porque `Test-NetConnection` nao confirma UDP.

## Observacoes

- Nao use GitHub/VS Code tunnel como caminho principal para conexao remota. Ele
  e bom para HTTP de desenvolvimento, mas o Stardesk precisa de portas TCP/UDP
  proprias para rendezvous e relay.
- Para producao, prefira DNS com Elastic IP em vez de IP dinamico.
- Restrinja SSH para o seu IP no Security Group.
- Mantenha a pasta `server-data` persistente; se ela for apagada, a chave publica do
  servidor muda e os clientes precisam ser reconfigurados.

## Referencias

- Docker Hub: `rustdesk/rustdesk-server`
- Documentacao RustDesk self-host Docker
