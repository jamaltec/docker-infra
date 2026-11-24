# JamalServer: ficha do servidor

Referência única do servidor de casa: peças, discos, rede, serviços e configurações feitas.
Atualizada em 24/11/2026. Ao mudar algo no servidor, atualize esta ficha no mesmo commit.

> Este repositório é **público**. Aqui não entram senhas, chaves de API nem tokens.
> Eles ficam em `/srv/docker/env_files/` no servidor, fora do git.

## Resumo

| Item | Valor |
| --- | --- |
| Nome | `JamalServer` |
| Sistema | Ubuntu 24.04 Desktop (GNOME, Wayland) |
| Usuário | `jamal` (UID 1000, GID 1000) |
| IP na rede de casa | `192.168.0.45` (placa `eno1`, cabeada) |
| Acesso SSH | `ssh jamal@JamalServer.local` ou `ssh jamal@192.168.0.45` |
| Gerenciador de containers | Komodo, com os composes deste repositório |
| Editor padrão | Helix (`hx`) |

## Hardware

| Peça | Detalhe | Observação |
| --- | --- | --- |
| CPU | Intel Core i5-4590 (4 núcleos, Haswell) | GPU integrada Intel HD 4600 |
| RAM | **a conferir** (`free -h`) | |
| SSD | 120 GB (`sdb`) | Sistema, Docker e configurações |
| HD 1 | 500 GB SATA (`sda`) | Pool de mídia |
| HD 2 | 500 GB SATA (`sdc`) | Pool de mídia |
| HD externo | WD Elements SE 1 TB, USB (`sdd`) | Fora do pool; uso a definir |
| Fonte | **modelo e potência a conferir** (etiqueta lateral) | Possível troca |
| Tela | Barra LCD 8,8" 1920×480, HDMI | Alimentar por fonte 5 V externa, não pela USB da placa-mãe |

**Sobre a GPU integrada:** a HD 4600 (Haswell) acelera H.264, mas **não decodifica HEVC 10-bit** por hardware.
Animes em HEVC 10-bit sempre convertem pela CPU, e isso causou o erro "Tempo limite excedido" no Jellyfin.

As letras `sda`, `sdb`... podem mudar entre boots. O que identifica cada disco é o UUID abaixo.

## Armazenamento

| Disco | Partição | Sistema de arquivos | Rótulo | UUID | Montado em |
| --- | --- | --- | --- | --- | --- |
| SSD | `sdb1` | vfat | | `14B5-44B5` | `/boot/efi` |
| SSD | `sdb2` | ext4 | | `134b884d-7463-4b6b-8daa-07824aa530b9` | `/` |
| HD 1 | `sda1` | ext4 | `hdd5002` | `0abe684e-91f7-4870-8e9f-a9fd3192e51a` | `/mnt/disco1` |
| HD 2 | `sdc1` | ext4 | `hdd5001` | `4ebd8d6a-b868-4948-8d76-6123d6677cce` | `/mnt/disco2` |
| HD externo | `sdd1` | ext4 | `disco3` | `25372ce7-b682-43ba-9be0-0efd9aa15436` | `/mnt/disco3` |

### Pool de mídia (mergerfs)

- `/srv/midia` junta `/mnt/disco1` e `/mnt/disco2` (cerca de 915 GB; 457 GB usados em 14/11/2026).
- Arquivo novo vai para o disco com mais espaço livre (`category.create=mfs`), deixando 20 GB livres em cada um (`minfreespace=20G`).
- Se um HD morrer, perde-se só o que estava nele. **Não há paridade nem backup da mídia.**
- Pastas: `filmes`, `series`, `animes`, `livros`, `downloads`.
- Os containers enxergam o pool como `/media` (por exemplo, `/media/filmes`).

Linhas do `/etc/fstab` (versão planejada; conferir com `cat /etc/fstab`):

```
UUID=0abe684e-91f7-4870-8e9f-a9fd3192e51a  /mnt/disco1  ext4  defaults,nofail,x-systemd.device-timeout=30  0 2
UUID=4ebd8d6a-b868-4948-8d76-6123d6677cce  /mnt/disco2  ext4  defaults,nofail,x-systemd.device-timeout=30  0 2
UUID=25372ce7-b682-43ba-9be0-0efd9aa15436  /mnt/disco3  ext4  defaults,nofail,x-systemd.device-timeout=30  0 2
/mnt/disco1:/mnt/disco2  /srv/midia  fuse.mergerfs  defaults,allow_other,cache.files=off,dropcacheonclose=true,category.create=mfs,moveonenospc=true,minfreespace=20G,fsname=mergerfs,nofail,x-systemd.requires-mounts-for=/mnt/disco1,x-systemd.requires-mounts-for=/mnt/disco2  0 0
```

Proteções:

- `chattr +i` nas pastas vazias `/mnt/disco1`, `/mnt/disco2` e `/mnt/disco3`: se um disco não montar, nada é gravado no SSD por engano.
- O Docker espera o pool montar (`RequiresMountsFor=/srv/midia` em `/etc/systemd/system/docker.service.d/override.conf`).
- Reserva de 5% do ext4 zerada nos HDs (`tune2fs -m 0`).

### Pastas do Docker (no SSD)

```
/srv/docker/
├── docker-infra/      este repositório (clonado no servidor)
│   └── stacks/compose/<app>/compose.yaml
├── env_files/         senhas e chaves de cada app (fora do git)
├── secrets/
└── volumes/<app>/     configuração e banco de cada app
```

## Serviços

Todos na rede Docker externa `media` (criada com `docker network create media`), em que um container chama o outro pelo nome.

| Serviço | Porta | Endereço na rede `media` | Função | Dados em |
| --- | --- | --- | --- | --- |
| Jellyfin | 8096 (e 7359/udp) | `http://jellyfin:8096` | Assistir | `volumes/jellyfin` |
| Seerr | 5055 | `http://seerr:5055` | Pedir filmes e séries | `volumes/seerr` |
| Radarr | 7878 | `http://radarr:7878` | Filmes | `volumes/radarr` |
| Sonarr | 8989 | `http://sonarr:8989` | Séries e animes | `volumes/sonarr` |
| Prowlarr | 9696 | `http://prowlarr:9696` | Indexadores | `volumes/prowlarr` |
| qBittorrent | 8080 (6881 tcp/udp) | `http://qbittorrent:8080` | Downloads, sem VPN por enquanto | `volumes/qbittorrent` |
| Homepage | 3000 | | Painel; **deploy pendente** | config neste repositório |
| Komodo | 9120 | | Deploy das stacks | `volumes/komodo` |

Pastas raiz: Radarr `/media/filmes`; Sonarr `/media/series` e `/media/animes`; qBittorrent salva em `/media/downloads`.
Quem trabalha nos arquivos de mídia roda como UID/GID 1000.

### Fluxo de deploy

1. Mudar o compose neste repositório.
2. `git pull` em `/srv/docker/docker-infra`.
3. Deploy da stack no Komodo e conferir os logs.

## Configurações do sistema

| O quê | Como ficou | Por quê |
| --- | --- | --- |
| Suspensão | `systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` | A tela na USB da placa-mãe enviava "Suspend key pressed" e o PC dormia |
| Tela não apaga | `gsettings` `idle-delay 0` e bloqueio de tela desligado | Telinha como monitor fixo |
| Telinha | Orientação girada nas Configurações; `btop` com `shown_boxes = "cpu net"` | A tela tem só 21 linhas de terminal |
| Editor | Helix como `editor` do sistema e `EDITOR`/`VISUAL`/`SYSTEMD_EDITOR` no `.bashrc` | Preferência |

## Acesso externo

- Domínio `midia.astrum.app.br` apontava para o Jellyfin; parou de responder depois do reboot de 14/11/2026. Origem a descobrir.
- Há uma VPN `wt0` ligada, provavelmente o NetBird (`netbird status` para confirmar).
- Planos: Oracle Cloud (plano grátis) como porta de entrada com túnel para casa, domínio `.com.br` próprio e proxy reverso com HTTPS. Alternativa simples: Tailscale.

## Outros aparelhos

| Aparelho | Detalhe | Papel |
| --- | --- | --- |
| PC principal | Windows, nome `desktop-h7roema`, RTX 4060 Ti, HD de 2 TB | Destino de backup; nó do Tdarr para AV1 no futuro |
| Tablet | Galaxy Tab S6 Lite | Painel fixo com o Homepage |

## Histórico

| Data | O que aconteceu |
| --- | --- |
| 14/11/2026 | Filmes copiados do HD externo (NTFS) para o pool; limpeza de duplicados e 720p; HD externo formatado em ext4 |
| 14/11/2026 | Reboot inesperado no meio da cópia; disco1 não montou no boot; proteções do fstab adicionadas |
| 15/11/2026 | Stacks de Sonarr, Radarr, Seerr, qBittorrent e Prowlarr; rede `media` |
| 16/11/2026 | Stack do Homepage |
| 17/11/2026 | Telinha de 8,8" instalada; suspensão causada pela tela na USB; suspensão bloqueada |

## Onde está o resto

- Guia de estudos e pendências do servidor: <https://claude.ai/artifact/PKNfMPh5K12RGXGvAL5KfQ>
- Setup físico (case da tela, fonte, tablet): <https://claude.ai/artifact/1R7LVNEXDNVssaZKg1uAsT>
- Lista de filmes e excluídos: <https://claude.ai/artifact/Nw5nkg38gWfcxh3MqKMPbh>
