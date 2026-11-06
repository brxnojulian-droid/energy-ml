# Despliegue en aprendizaje-automatico.iftn199.com.ar

Destino: VPS `vps` (alias de `~/.ssh/config`, AlmaLinux 8), detrás de Cloudflare (modo Flexible)
y nginx en el origen. uvicorn escucha solo en `127.0.0.1:8020`.
Modo de datos: `local` (CSV en `data/`). El modo `ssh` no se usa en el sitio público.

Archivos de referencia:
- `deploy/energy-ml.service` → unidad systemd (puerto 8020)
- `deploy/nginx-aprendizaje-automatico.conf` → vhost nginx (solo :80, como `dev.isftn199.com.ar`)

## 1. DNS en Cloudflare (lo carga quien administra la zona iftn199.com.ar)

Registro DNS en la zona `iftn199.com.ar`:

```
aprendizaje-automatico   A   168.181.184.103   (proxied: naranja)
```

Verificar: `dig +short aprendizaje-automatico.iftn199.com.ar`

## 2. Código en el VPS

```bash
sudo useradd --system --home /opt/energy-ml --shell /usr/sbin/nologin energy-ml
sudo git clone --branch <RAMA_DE_DESPLIEGUE> https://github.com/datamaq-automation/energy-ml /opt/energy-ml
cd /opt/energy-ml
sudo python3 -m venv .venv
sudo .venv/bin/pip install -r requirements.txt
```

Datos: `data/input/` y `data/verdad/` se copian desde la máquina de desarrollo:

```bash
rsync -av data/input/ vps:/opt/energy-ml/data/input/
rsync -av data/verdad/ vps:/opt/energy-ml/data/verdad/
```

Permisos: `sudo chown -R energy-ml:energy-ml /opt/energy-ml/data`

## 3. Servicio

```bash
sudo cp deploy/energy-ml.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now energy-ml
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8020/   # esperar 200
```

## 4. nginx

```bash
sudo cp deploy/nginx-aprendizaje-automatico.conf /etc/nginx/conf.d/aprendizaje-automatico.iftn199.com.ar.conf
sudo nginx -t && sudo systemctl reload nginx
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: aprendizaje-automatico.iftn199.com.ar" -H "X-Forwarded-Proto: https" http://127.0.0.1/
```

No hace falta certbot: el TLS termina en Cloudflare.

## 5. Verificación

```bash
curl -I https://aprendizaje-automatico.iftn199.com.ar/
curl -I https://aprendizaje-automatico.iftn199.com.ar/guia
```

## Pendiente antes de exponer al público

- `ALLOWED_HOSTS` está en `["*"]` (`src/infrastructure/settings/config.py`). Conviene limitarlo al subdominio.
- `/docs` (OpenAPI) queda público. Decidir si se mantiene.
- Subir CSV desde la web: fuera de alcance por ahora.
