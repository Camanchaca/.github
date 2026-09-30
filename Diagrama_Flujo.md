# Diagrama de Flujo: Procedimiento de Cambios en Infraestructura y Software

### 1. Flujo General de Solicitud y Evaluación de Cambios

```mermaid
graph TD
    A([Inicio: Propuesta de Cambio]) --> B[Desarrolladores/Arquitectos: Solicitan cambio a Líder TI Divisional]
    B --> C{¿Impacto Medio o Alto?}
    
    C -- No (Cambio Estándar) --> D[Registrar e implementar cambio estándar]
    C -- Sí (Cambio Normal) --> E[Adjuntar Planificación, Rollback y Ventana de Menor Impacto]
    
    E --> F[Líder TI Divisional: Presenta propuesta al Comité de Cambios]
    F --> G[Comité de Cambios: Evalúa alcance, impacto y factibilidad]
    
    G --> H{¿Aprobado por el Comité?}
    H -- No --> I([Fin: Cambio Rechazado])
    H -- Sí --> J{¿Corresponde a Desarrollo de Software?}
    
    J -- No (Solo Infraestructura) --> K[Arquitectos/TI: Implementan cambio según lo acordado]
    K --> L([Fin: Cambio Implementado])
    
    J -- Sí (Software/BD) --> M[Equipo TI: Proporciona GitHub Org, Repo base, Sandbox temporal y GCP Prod]
    M --> N[Ver Diagrama 2: Flujo de Software]
```

---

### 2. Flujo de Desarrollo, Pruebas y Despliegue de Software

```mermaid
graph TD
    A([Inicio: Repositorio e Infraestructura Proporcionados]) --> B{¿Tipo de Desarrollo?}
    
    B -- Nuevo Desarrollo --> C[Desarrolladores: Usar rama 'develop' y proyecto Sandbox temporal]
    B -- Prueba / Actualización --> D[Desarrolladores: Usar rama 'qas' e instancia QAS en GCP Prod]
    
    C --> E[Verificar Skill SecOps Vibe Coding en .agents/harness/skills]
    D --> E
    
    E --> F[Desarrollar y probar usando EXCLUSIVAMENTE datos ficticios/sintéticos]
    F --> G[Realizar Commits locales sin credenciales expuestas]
    
    G --> H[Generar Pull Request desde 'develop' o 'qas' hacia 'main']
    H --> I[Comité de Cambios: Identifica y revisa el Pull Request]
    
    I --> J{¿Aprobado por Líder TI Divisional Y Miembro Corporativo?}
    J -- No --> K[Notificar motivos de rechazo al equipo desarrollador]
    K --> F
    
    J -- Sí --> L{¿Contiene Base de Datos?}
    L -- Sí --> M[Coordinar migración de BD con Equipo TI antes de desplegar código]
    L -- No --> N[Fusión a 'main' e inicio de Pipeline CI/CD automatizado]
    M --> N
    
    N --> O[Ejecución de Guardianes CI/CD: Gitleaks, Trivy, Semgrep, SCA]
    O --> P{¿Pasa los 4 Guardianes de Seguridad?}
    
    P -- No --> Q[Pipeline Interrumpido: Desarrollador corrige vulnerabilidades]
    Q --> F
    
    P -- Sí --> R[Despliegue Automático a Producción en GCP]
    R --> S[Equipo TI: Da de baja el entorno Sandbox temporal]
    S --> T([Fin: Software Desplegado en Producción])
```