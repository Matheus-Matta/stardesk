# Stardesk

Stardesk e um fork rebrandado do RustDesk para suporte remoto. O objetivo do
fork e manter a base upstream atualizavel, trocar identidade visual/nome e
preparar builds proprios do cliente.

Este projeto herda a licenca AGPL-3.0 do RustDesk. Antes de distribuir ou usar
uma versao modificada em rede, leia [LICENCE](LICENCE) e
[docs/STARDESK_REBRAND.md](docs/STARDESK_REBRAND.md).

## Estado do rebrand

- Nome do app: `Stardesk`
- Esquema de URI: `stardesk://`
- Bundle/application ID: `com.stardesk.app`
- Binario desktop Flutter: `stardesk`
- Cor principal inicial: `#0058BB`

Ainda falta substituir os assets finais de marca em `res/icon.png` e
`res/mac-icon.png`.

## Requisitos

No Windows:

- Git
- Python 3
- Rust/Cargo via rustup
- Flutter
- Visual Studio com workload "Desktop development with C++"
- LLVM/Clang
- vcpkg configurado em `VCPKG_ROOT`
- Modo de Desenvolvedor do Windows habilitado para symlinks do Flutter

Para abrir a tela do Modo de Desenvolvedor:

```powershell
start ms-settings:developers
```

Instale as dependencias nativas principais no vcpkg:

```powershell
vcpkg install libvpx:x64-windows-static libyuv:x64-windows-static opus:x64-windows-static aom:x64-windows-static
```

## Preparar o repo

```powershell
git clone --recurse-submodules https://github.com/Matheus-Matta/stardesk.git
cd stardesk
git checkout rebrand/stardesk
git submodule update --init --recursive
```

Se voce ja clonou sem submodulos:

```powershell
git submodule update --init --recursive
```

## Executar em desenvolvimento

Use a ordem abaixo para validar o app: Windows desktop primeiro, Web depois, e
Mobile quando o SDK/dispositivo estiver configurado.

### 1. Windows desktop

Use este fluxo para desenvolver a interface Flutter com o core Rust local.

Primeiro prepare as dependencias uma vez:

```powershell
$env:VCPKG_ROOT="C:\vcpkg"
$env:VCPKG_DEFAULT_TRIPLET="x64-windows-static"
$env:LIBCLANG_PATH="C:\Program Files\LLVM\bin"
$env:RUST_LOG="info"

cd flutter
flutter config --enable-windows-desktop
flutter pub get
cd ..

flutter_rust_bridge_codegen --rust-input .\src\flutter_ffi.rs --dart-output .\flutter\lib\generated_bridge.dart --class-name Rustdesk
```

Depois compile o core Rust e execute o app:

```powershell
$env:VCPKG_ROOT="C:\vcpkg"
$env:VCPKG_DEFAULT_TRIPLET="x64-windows-static"
$env:LIBCLANG_PATH="C:\Program Files\LLVM\bin"

cargo build --features flutter --lib

cd flutter
flutter run -d windows
```

Se o `flutter run` compilar mas perder a conexao com a janela, rode o binario
Debug diretamente:

```powershell
.\build\windows\x64\runner\Debug\stardesk.exe --noinstall
```

Se o Flutter reclamar de symlinks, habilite o Modo de Desenvolvedor do Windows
e rode `flutter pub get` novamente.

### 2. Web

O projeto tem pasta `flutter/web` e uma ponte web em `flutter/lib/web/bridge.dart`.
Esse modo serve para validar tela/fluxos de UI; ele nao substitui o app desktop
completo com core Rust nativo.

```powershell
cd flutter
flutter config --enable-web
flutter pub get
flutter run -d chrome
```

Para listar outros destinos web disponiveis:

```powershell
flutter devices
```

### 3. Mobile

O projeto tem pastas `flutter/android` e `flutter/ios`.

Android, com Android Studio/SDK e um emulador ou aparelho conectado:

```powershell
cd flutter
flutter pub get
flutter devices
flutter run -d android
```

iOS so compila em macOS com Xcode configurado:

```sh
cd flutter
flutter pub get
flutter devices
flutter run -d ios
```

### Build rapido sem empacotar

Este comando gera uma build release do app Flutter, mas pula o pacote portatil.
E o melhor primeiro teste de build em maquina nova.

```powershell
python build.py --flutter --skip-portable-pack
```

Saida esperada no Windows:

```text
flutter/build/windows/x64/runner/Release/stardesk.exe
```

### Core Rust sem Flutter

O cliente Sciter e legado, mas ainda pode ajudar a validar ambiente Rust/C++.
No Windows, baixe o `sciter.dll` e coloque em `target/debug`.

```powershell
mkdir target\debug
curl.exe -L https://raw.githubusercontent.com/c-smile/sciter-sdk/master/bin.win/x64/sciter.dll -o target\debug\sciter.dll
cargo run
```

## Gerar build de producao

### GitHub Actions

Use a workflow `Stardesk build` em `Actions` para gerar:

- artefato Windows em `stardesk-windows-x64`;
- artefato Web em `stardesk-web`;
- imagem Docker do servidor em `ghcr.io/matheus-matta/stardesk-server`.

A workflow tambem roda automaticamente em tags `v*`.

### Windows release com Flutter

```powershell
python build.py --flutter
```

Esse fluxo compila o core Rust, gera o app Flutter em release e cria o pacote
portatil/instalador quando o empacotamento estiver habilitado. Para testar sem
empacotar, use:

```powershell
python build.py --flutter --skip-portable-pack
```

### Linux release com Flutter

Instale as dependencias de sistema equivalentes ao RustDesk upstream, configure
`VCPKG_ROOT`, depois rode:

```sh
python3 build.py --flutter
```

### macOS release com Flutter

Configure Rust, Flutter, Xcode/Command Line Tools e as dependencias nativas.
Depois rode:

```sh
python3 build.py --flutter
```

Assinatura, notarizacao e empacotamento final exigem certificados Apple
proprios.

## Regenerar icones

Depois de trocar `res/icon.png` e `res/mac-icon.png`:

```powershell
cd flutter
flutter pub get
flutter pub run flutter_launcher_icons
```

## Servidor proprio

O servidor RustDesk OSS fica em outro repositorio. Para usar infraestrutura
propria do Stardesk:

Para subir em VPS/AWS com Docker, veja
[docs/AWS_DOCKER_SERVER.md](docs/AWS_DOCKER_SERVER.md).

O compose pronto fica em `docker-compose.server.yml`; copie
`.env.server.example` para `.env.server` e ajuste `STARDESK_SERVER_HOST`.

```sh
git clone https://github.com/rustdesk/rustdesk-server.git stardesk-server
cd stardesk-server
cargo build --release
```

Os binarios principais gerados sao `hbbs`, `hbbr` e `rustdesk-utils`.

## Atualizar com o upstream

```powershell
git checkout master
git pull upstream master

git checkout rebrand/stardesk
git merge master
```

Resolva conflitos mantendo a camada de rebrand pequena e separada.

## Problemas comuns

- `cargo` nao encontrado: instale o Rust via rustup e reabra o terminal.
- `flutter pub get` falha por symlink: habilite o Modo de Desenvolvedor do Windows.
- Android SDK nao encontrado: instale Android Studio ou configure `flutter config --android-sdk`.
- Erro de dependencias C++: confirme Visual Studio C++, LLVM e `VCPKG_ROOT`.

## Licenca e uso responsavel

Stardesk e baseado no RustDesk e segue AGPL-3.0. Preserve avisos legais,
disponibilize o codigo-fonte correspondente quando aplicavel e use a ferramenta
apenas com consentimento explicito do cliente/usuario.
