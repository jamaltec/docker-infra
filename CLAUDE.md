# docker-infra

Composes do servidor de casa `JamalServer`, gerenciados pelo Komodo.

Antes de mexer em qualquer coisa do servidor, leia `docs/SERVIDOR.md`: discos, UUIDs, rede, portas, serviços e o que já foi configurado.
Ao mudar algo no servidor, atualize `docs/SERVIDOR.md` no mesmo commit.

- Um app por pasta em `stacks/compose/<app>/compose.yaml`.
- Dados em `/srv/docker/volumes/<app>/`, mídia em `/srv/midia` (montada como `/media`), rede externa `media`, UID/GID 1000.
- O repositório é público: senhas e chaves de API ficam em `/srv/docker/env_files/`, nunca aqui.
- O dono do servidor está aprendendo Linux: explique comandos em português e peça confirmação antes de `rm`, `mkfs`, `chown -R` ou mudanças no `/etc/fstab`.
