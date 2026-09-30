# Reglas Corporativas de Ciberseguridad & DevSecOps (Camanchaca SecOps)

Estas reglas son de cumplimiento obligatorio para todos los desarrollos de software, arquitecturas y agentes de IA en Camanchaca:

> **ACTIVACIÓN OBLIGATORIA DEL SKILL:**
> Para diseñar, implementar, auditar, refactorizar o desplegar pipelines DevSecOps, contenedores Docker, configuraciones de Nginx, seguridad en la nube (GCP) o gobernanza de secretos, **DEBES activar y consultar obligatoriamente el skill `camanchaca-secops-standard`**.

## 1. Gobernanza de Secretos (Zero-Hardcoded Secrets)
- **PROHIBIDO:** Usar Base64, ofuscación o strings reversibles como pretendida protección de credenciales o API keys (CWE-312 / CWE-565).
- **MANDATORIO SERVIDOR/CI-CD:** Toda credencial sensible (passwords, llaves privadas, API keys de backend) debe inyectarse en tiempo de ejecución vía **Google Cloud Secret Manager** (`--set-secrets`) o **Workload Identity Federation (WIF)**.
- **IDENTIFICADORES PÚBLICOS FRONTEND:** El `CLIENT_ID` de OAuth 2.0 y `apiKey` de Firebase deben tratarse como identificadores públicos estándar (RFC 6749) y protegerse obligatoriamente mediante:
  1. Restricciones de Origen (HTTP Referrer) en Google Cloud Console.
  2. Aislamiento estricto de APIs en GCP IAM.
  3. Validación de identidad y claims en el servidor vía **Firestore Security Rules**.

## 2. Pipeline de Seguridad DevSecOps (5 Capas Mandatorias)
Todo despliegue a producción debe superar sin excepciones los 5 guardianes de seguridad:
1. **Paso 0a (Gitleaks):** Escaneo estricto de secretos y tokens en el 100% de los archivos.
2. **Paso 0b (Trivy Config):** Auditoría de malas configuraciones IaC (Dockerfile/Nginx) contra CIS Benchmarks.
3. **Paso 0c (Semgrep SAST):** Análisis estático de código JavaScript / Python / Go contra OWASP Top 10.
4. **Paso 2b (SBOM CycloneDX):** Generación automática de inventario criptográfico de componentes `secops-sbom.cdx.json`.
5. **Paso 3 (Trivy SCA):** Bloqueo automático ante vulnerabilidades del Sistema Operativo con severidad CRITICAL o HIGH.

## 3. Hardening de Servidor Web (Nginx)
Todo contenedor que sirva interfaces web debe implementar:
- `Content-Security-Policy` estricta (A+ rating).
- `Strict-Transport-Security` (HSTS: `max-age=63072000; includeSubDomains; preload`).
- `X-Frame-Options: DENY` (Anti-Clickjacking).
- `X-Content-Type-Options: nosniff`.
- `Permissions-Policy: geolocation=(), camera=(), microphone=()`.
- Ejecución con usuario sin privilegios y supresión de `server_tokens off;`.

## 4. Seguridad de Frontend & DOM
- Toda inserción de datos dinámicos en el DOM debe sanitizarse mediante funciones de escape (`escapeHtml`) para prevenir XSS (Cross-Site Scripting).
- Las sesiones deben usar un patrón **Zero-Flicker Session Gate** para evitar parpadeos y evaluar el estado de login antes del renderizado de la UI.
