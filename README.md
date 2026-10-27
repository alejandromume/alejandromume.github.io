# Blog de Lava Digital

Blog con Jekyll (GitHub Pages) y tema propio oscuro con la estética de Lava Digital.

## Desarrollo local
    bundle install
    bundle exec jekyll serve

## Nueva entrada
Crea `_posts/AAAA-MM-DD-titulo.md`:

    ---
    title: "Título"
    severity: info   # opcional, solo para entradas de seguridad: critical | high | medium | low | info
    cvss: "7.5"      # opcional
    scope: "Alcance" # opcional
    tags: [web, owasp]
    ---

Datos de la empresa y enlaces: `_config.yml` (bloque `author`).
