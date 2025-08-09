Yönetim (parent’a dokunmadan)

Sadece bir servisi durdur:
docker compose -f comfyui.yaml stop comfyui

Sadece bir servisi kaldır (container sil):
docker compose -f comfyui.yaml rm -f comfyui

log bak:
docker compose -f open-webui.yaml logs -f
