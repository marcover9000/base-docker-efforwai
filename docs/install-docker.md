# Instal·lar Docker (Debian Bookworm)

Guia per posar Docker Engine + Compose v2 a una màquina de
desenvolupament. Executa-ho com el teu usuari (els que necessiten
root porten `sudo`).

## 1. Netejar paquets antics (per si de cas)

```bash
sudo apt remove docker docker-engine docker.io containerd runc 2>/dev/null
```

## 2. Prerequisits i clau GPG del repo oficial

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

## 3. Afegir el repo de Docker

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 4. Instal·lar Docker Engine + Compose v2 + Buildx

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 5. Habilitar el servei i afegir-te al grup `docker`

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

> **Important:** has de tancar sessió i tornar a entrar (o reiniciar)
> perquè el grup `docker` s'apliqui. Si no, hauràs d'usar `sudo`
> davant de cada `docker ...`.

## 6. Comprovar

```bash
docker run --rm hello-world
docker compose version
```

## 7. (Opcional) Portainer CE

UI web per gestionar contenidors, imatges i volums.

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=always \
  -p 9443:9443 \
  -p 8000:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Després obre al navegador: <https://localhost:9443>

Et demanarà crear l'usuari admin (contrasenya mínim 12 caràcters) i
triar l'entorn → escull **Get Started / Local** (queda connectat al
socket del sistema).

A partir d'aquí, qualsevol `docker run`, `docker compose up`, etc.
que facis per terminal apareixerà al panell de Portainer en segons.
