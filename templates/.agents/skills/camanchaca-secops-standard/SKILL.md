---
name: camanchaca-secops-standard
description: >-
  Estándar oficial corporativo de DevSecOps, Hardening y Seguridad Cloud de Camanchaca.
  Utiliza este skill para diseñar, implementar, auditar o replicar la arquitectura de 5 niveles de DevSecOps:
  Gitleaks (Secret Scanning), Trivy (IaC & Container SCA), Semgrep (SAST OWASP Top 10),
  CycloneDX (SBOM Supply Chain), Nginx Hardening (CSP A+, HSTS), Secret Manager / WIF,
  y Local Security Test Harness (Shift-Left).
---

# Estándar Corporativo de Arquitectura Segura y DevSecOps (Camanchaca)

Este Skill documenta la arquitectura de referencia y los procedimientos obligatorios para garantizar que cualquier proyecto de software en la compañía cumpla con los 15 controles de seguridad y el pipeline DevSecOps de 5 capas.

---

## 0. Perímetro Zero-Trust de Acceso (Load Balancer + IAP)

Todo servicio corporativo en Cloud Run debe estar protegido por el perímetro de red de Google Cloud:
1. **External Application Load Balancer:** Proporciona terminación TLS centralizada, mitigación DDoS y certificado gestionado por Google.
2. **Identity-Aware Proxy (IAP):** Exige autenticación OIDC previa con cuenta corporativa `@camanchaca.cl` y 2FA antes de servir cualquier byte de la aplicación.
3. **Cloud Run Ingress Protection:** Debe configurarse en `internal-and-cloud-load-balancing` para bloquear accesos directos por la URL `*.a.run.app`.
4. *(Opcional / Roadmap)* **Cloud Armor:** Política de WAF y lista blanca de IPs para restringir el acceso a nivel de red hacia sedes y VPN.

---

## 1. Los 5 Guardianes del Pipeline DevSecOps (Cloud Build)

Todo proyecto debe incluir un archivo `cloudbuild.yaml` estructurado con los 5 filtros secuenciales:

```yaml
steps:
  # 1. Guardián de Secretos (Gitleaks)
  - name: 'zricethezav/gitleaks:latest'
    id: 'source-code-secret-scan'
    args: ['detect', '--source=/workspace', '--verbose', '--exit-code=1']

  # 2. Guardián de Malas Configuraciones IaC (Trivy Config)
  - name: 'aquasec/trivy:latest'
    id: 'iac-misconfiguration-scan'
    args: ['config', '--exit-code', '1', '--severity', 'CRITICAL', '/workspace']

  # 3. Guardián SAST de Lógica de Código (Semgrep OWASP Top 10)
  - name: 'semgrep/semgrep:latest'
    id: 'sast-code-logic-scan'
    args: ['semgrep', 'scan', '--config=p/javascript', '--config=p/owasp-top-ten', '--error', '--metrics=off', '/workspace/js']

  # 4. Compilación y Push de Imagen de Contenedor
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    args: ['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:$COMMIT_SHA', '.']

  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    args: ['push', 'us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:$COMMIT_SHA']

  # 5. Guardián de Cadena de Suministro: Generación de SBOM (CycloneDX Standard)
  - name: 'aquasec/trivy:latest'
    id: 'generate-sbom-cyclonedx'
    args: ['image', '--format', 'cyclonedx', '--output', '/workspace/secops-sbom.cdx.json', 'us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:$COMMIT_SHA']

  # 6. Guardián SCA de Vulnerabilidades en SO (Trivy Image)
  - name: 'aquasec/trivy:latest'
    id: 'container-vulnerability-scan'
    args: ['image', '--exit-code', '1', '--severity', 'CRITICAL,HIGH', '--ignore-unfixed', 'us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:$COMMIT_SHA']

  # 7. Despliegue Seguro a Google Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy-to-cloud-run'
    entrypoint: 'gcloud'
    args: ['run', 'deploy', 'service-name', '--image', 'us-central1-docker.pkg.dev/$PROJECT_ID/repo/app:$COMMIT_SHA', '--region', 'us-central1', '--platform', 'managed']
```

---

## 2. Hardening Obligatorio de Servidores Web (Nginx)

El archivo `nginx.conf` debe contar con las siguientes directivas de seguridad para alcanzar calificación de seguridad A+:

```nginx
server {
    listen 8080;
    server_tokens off;

    # Cabeceras de Seguridad HTTP Estrictas
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://accounts.google.com https://www.gstatic.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com data:; connect-src 'self' https://firestore.googleapis.com https://accounts.google.com; frame-src https://accounts.google.com;" always;

    # Compresión Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
}
```

---

## 3. Política Cero Secretos en Código (Secret Governance)

1. **Prohibición de Base64:** Ninguna credencial ni token debe ofuscarse en Base64 bajo la presunción de seguridad.
2. **Secretos de Backend / Servidor:** Deben almacenarse en **Google Cloud Secret Manager** e inyectarse en el contenedor de Cloud Run mediante variables de entorno en tiempo de ejecución:
   ```bash
   gcloud run deploy mi-servicio --set-secrets="DB_PASS=db-password:latest,API_TOKEN=api-token:latest"
   ```
3. **Identificadores Frontend:** `CLIENT_ID` y `apiKey` son identificadores públicos (RFC 6749) protegidos mediante **Restricciones de Origen HTTP Referrer** y **Firestore Security Rules**.

---

## 4. Local Shift-Left Security Test Harness (`security-harness.ps1`)

Todo repositorio debe proveer un script de auditoría local para que los desarrolladores validen su código antes de hacer commit:
- **Test 1:** `gitleaks detect --source=. --no-git`
- **Test 2:** `trivy config .`
- **Test 3:** `semgrep scan --config=p/javascript --config=p/owasp-top-ten js`
- **Test 4:** `trivy image` + Generación local de `secops-sbom.cdx.json`.
