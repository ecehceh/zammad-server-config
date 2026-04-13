# Folder ini untuk menyimpan sertifikat SSL Traefik.
# File .crt dan .key TIDAK di-push ke GitHub (lihat .gitignore).

# Generate Self-Signed Certificate (untuk development/testing):
# ─────────────────────────────────────────────────────────────
# Linux/Mac:
#   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
#     -keyout zammad.key -out zammad.crt \
#     -subj "/CN=172.21.3.99"
#
# Windows (PowerShell / Git Bash):
#   openssl req -x509 -nodes -days 365 -newkey rsa:2048 `
#     -keyout zammad.key -out zammad.crt `
#     -subj "/CN=172.21.3.99"
#
# Ganti 172.21.3.99 dengan domain atau IP server Anda.
