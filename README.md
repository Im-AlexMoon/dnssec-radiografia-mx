# Radiografía de DNSSEC bajo el dominio .mx

Análisis de la adopción y salud criptográfica de DNSSEC en dominios representativos del ecosistema mexicano, realizado como parte del bloque **Análisis de Criptografía y Seguridad (MA2002B)** en el Tecnológico de Monterrey, Campus Monterrey.

**Socio Formador:** NIC México  
**Equipo 2 · Grupo 302**  
Junio 2026

---

## Contexto

DNSSEC (DNS Security Extensions) es el mecanismo estándar para proteger al DNS contra ataques de suplantación de dominio. Aunque NIC México mantiene correctamente firmado el TLD `.mx`, la protección no se hereda automáticamente: cada titular de dominio debe activarla de forma independiente.

Este proyecto evalúa nueve dominios bajo `.mx` —gubernamentales, educativos, financieros y de registro— para determinar cuáles cuentan con protección DNSSEC funcional, cuáles presentan configuraciones rotas y cuáles carecen por completo de la extensión.

## Dominios analizados

| Dominio | Sector | Estado DNSSEC |
|---------|--------|---------------|
| org.mx | Registro (NIC México) | ✅ Completo |
| gob.mx | Registro (NIC México) | ✅ Completo |
| com.mx | Registro (NIC México) | ✅ Completo |
| unam.mx | Educativo | ✅ Completo |
| banxico.org.mx | Financiero | ⚠️ DS huérfano |
| gbm.org.mx | Financiero | ⚠️ DS huérfano |
| tec.mx | Educativo | ❌ Sin DNSSEC |
| sat.gob.mx | Gubernamental | ❌ Sin DNSSEC |
| udlap.mx | Educativo | ❌ DS desalineado |

## Resultados principales

- **4 de 9** dominios (44%) tienen DNSSEC completo con cadena de confianza íntegra desde la raíz.
- **2 dominios** presentan el patrón de DS huérfano: el padre publica un DS pero la zona hija no responde con DNSKEY.
- **3 dominios** carecen de DNSSEC funcional, incluyendo sat.gob.mx, dominio del SAT que maneja información fiscal de millones de contribuyentes.
- El algoritmo predominante es RSA/SHA-256 (Algoritmo 8). Solo unam.mx utiliza ECDSA P-256/SHA-256 (Algoritmo 13), actualmente el más recomendado por el RFC 8624.
- 3 de 4 dominios firmados usan NSEC3 (RFC 5155); unam.mx usa NSEC tradicional.

## Estructura del repositorio

```
├── README.md                  ← Este archivo
├── data/
│   └── resultados_dnssec.csv  ← Tabla consolidada de resultados por dominio
├── scripts/
│   └── dnssec_audit.py        ← Script de consulta automatizada y generación de CSV
├── diagramas/
│   ├── flujo_validacion_dnssec.png  ← Diagrama del flujo de validación DNSSEC
│   └── arboles/               ← Árboles de cadena de confianza por dominio (generados con graphviz)
├── capturas/                  ← Capturas dig/Wireshark de evidencia
└── docs/
    └── Reporte_Tecnico_DNSSEC_Equipo2.pdf
```

## Uso del script

El script `dnssec_audit.py` automatiza las consultas DNSSEC para cada dominio de la lista y genera el archivo CSV con los resultados.

### Dependencias

```
pip install dnspython graphviz
```

### Ejecución

```bash
# Generar CSV con resultados de todos los dominios
python scripts/dnssec_audit.py

# Generar CSV + árboles de cadena de confianza
python scripts/dnssec_audit.py --arboles
```

El script consulta los registros DNSKEY, RRSIG, DS, NSEC y NSEC3PARAM de cada dominio en la jerarquía completa (raíz → TLD → dominio objetivo), verifica la correspondencia DS ↔ DNSKEY mediante reconstrucción del digest, y clasifica cada dominio según los criterios de la Tabla 1 del reporte técnico.

### Salida

- `data/resultados_dnssec.csv` — tabla con columnas: Dominio, DNSSEC activo, Valida, Motivo de fallo, NSEC/NSEC3, Algoritmo, Keysize (bits), RRset firmadas, DS en zona padre, Observaciones.
- `diagramas/arboles/*.png` — representación gráfica de la cadena de confianza de cada dominio (si se usa `--arboles`).

## Metodología

1. **Consultas DNS públicas** con `dig +dnssec` dirigidas a resolvers públicos (1.1.1.1) para obtener registros DNSKEY, RRSIG, DS, NSEC/NSEC3PARAM.
2. **Captura de tráfico** con Wireshark durante sesiones de resolución activa para observar los registros en contexto de red.
3. **Verificación criptográfica** automatizada: reconstrucción del digest del DNSKEY y comparación con el DS publicado por la zona padre.
4. **Clasificación** en tres niveles: DNSSEC completo, DNSSEC parcial (DS huérfano / DS desalineado) y Sin DNSSEC, verificando el flag AD en la respuesta.

Para detalles completos, consultar el reporte técnico en `docs/`.

## Flujo de validación DNSSEC

```
   ┌──────────┐
   │   Raíz   │  KSK/ZSK propias, DS de .mx publicado
   │    (.)   │
   └────┬─────┘
        │ DS(.mx) ──→ coincide con DNSKEY(.mx)?
        ▼
   ┌──────────┐
   │   .mx    │  KSK/ZSK (KeyTag 43850/26638), DS de zonas hijas
   │ NIC Méx. │
   └────┬─────┘
        │ DS(dominio) ──→ coincide con DNSKEY(dominio)?
        ▼
   ┌──────────┐
   │ dominio  │  DNSKEY + RRSIG propios → cadena íntegra
   │  .mx     │  Sin DNSKEY → cadena rota (DS huérfano)
   └──────────┘  Sin DS en padre → sin DNSSEC
```

Cada eslabón se verifica de forma independiente. La cadena es tan fuerte como su eslabón más débil.

## Marco normativo

| RFC | Descripción |
|-----|-------------|
| 4033 | DNS Security Introduction and Requirements |
| 4034 | Resource Records for DNSSEC (DNSKEY, RRSIG, DS, NSEC) |
| 4035 | Protocol Modifications for DNSSEC |
| 5155 | NSEC3 — Hashed Authenticated Denial of Existence |
| 6781 | DNSSEC Operational Practices, Version 2 |
| 8624 | Algorithm Implementation Requirements for DNSSEC |
| 9364 | DNS Security Extensions (DNSSEC) — estado del arte 2023 |

## Herramientas de IA

Se utilizó ChatGPT (OpenAI) como apoyo para la generación, revisión y depuración de fragmentos de código del script de árboles DNSSEC y el CSV. También se empleó como apoyo en la redacción del reporte técnico. Todo el contenido técnico, hallazgos, clasificaciones y conclusiones fueron generados por el equipo a partir de datos reales. Los detalles se encuentran en el Anexo A del reporte técnico.

## Autores

| Nombre | Matrícula |
|--------|-----------|
| Alberto José Corella Moreno | A01255649 |
| Enrique Alexander Luna Sánchez | A00574602 |
| Diego Villalón Aguilar | A00842152 |
| Fausto Armando Orozco Kuri | A01613373 |
| Daniel Alberto Ayala | A01199321 |
| Diego Esparza Guzmán | A00841105 |
