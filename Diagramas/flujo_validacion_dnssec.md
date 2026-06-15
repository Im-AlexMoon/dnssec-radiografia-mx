# Flujo de validación DNSSEC (raíz → TLD → dominio)

```mermaid
flowchart TD
    subgraph RAIZ["Zona Raíz (.)"]
        R_KSK["KSK raíz\n(ancla de confianza)"]
        R_ZSK["ZSK raíz"]
        R_DS["DS de .mx\npublicado en raíz"]
        R_KSK --> R_ZSK
        R_ZSK -.->|firma| R_DS
    end

    subgraph MX["Zona .mx (NIC México)"]
        MX_DNSKEY["DNSKEY .mx\nKSK KeyTag 43850\nZSK KeyTag 26638\nAlgoritmo 8 RSA/SHA-256"]
        MX_RRSIG["RRSIG .mx\nfirma los RRsets\nde la zona"]
        MX_DS["DS del dominio hijo\npublicado en .mx"]
        MX_NSEC3["NSEC3PARAM\nSalt: 83935CE69BF387BD"]
        MX_DNSKEY --> MX_RRSIG
        MX_RRSIG -.->|firma| MX_DS
    end

    subgraph HIJO["Zona del dominio hijo"]
        H_DNSKEY["DNSKEY del dominio\n(KSK + ZSK propias)"]
        H_RRSIG["RRSIG del dominio\nfirma sus propios RRsets"]
        H_NSEC["NSEC / NSEC3\ndenegación de existencia"]
        H_DNSKEY --> H_RRSIG
        H_RRSIG -.->|firma| H_NSEC
    end

    R_DS -->|"① Verificar:\nhash(DNSKEY .mx) = DS en raíz?"| MX_DNSKEY
    MX_DS -->|"② Verificar:\nhash(DNSKEY hijo) = DS en .mx?"| H_DNSKEY

    subgraph RESULTADO["Resultado de la validación"]
        direction LR
        OK["✅ Cadena íntegra\nFlag AD = 1\n(org.mx, gob.mx,\ncom.mx, unam.mx)"]
        HUERFANO["⚠️ DS huérfano\nDS en padre, sin DNSKEY en hijo\n(banxico.org.mx,\ngbm.org.mx)"]
        ROTO["❌ Sin DNSSEC\nSin registros criptográficos\n(tec.mx, sat.gob.mx,\nudlap.mx)"]
    end

    H_DNSKEY -->|"③ Si ambas\nverificaciones pasan"| OK
    MX_DS -->|"③ Si hijo no responde\ncon DNSKEY"| HUERFANO
    MX_DS -->|"③ Si no existe DS\no no hay registros"| ROTO

    style RAIZ fill:#1a1a2e,color:#e0e0e0,stroke:#16213e
    style MX fill:#16213e,color:#e0e0e0,stroke:#0f3460
    style HIJO fill:#0f3460,color:#e0e0e0,stroke:#533483
    style OK fill:#2e7d32,color:#ffffff,stroke:#1b5e20
    style HUERFANO fill:#e65100,color:#ffffff,stroke:#bf360c
    style ROTO fill:#b71c1c,color:#ffffff,stroke:#7f0000
    style RESULTADO fill:#fafafa,color:#333,stroke:#ccc
```

## Lectura del diagrama

La validación DNSSEC se realiza eslabón por eslabón, de arriba hacia abajo:

1. **Raíz → .mx:** El resolver toma el registro DS de `.mx` publicado en la zona raíz y verifica que su hash coincida con el DNSKEY de la zona `.mx`. Si coincide, el primer eslabón es válido.

2. **.mx → dominio hijo:** El resolver toma el DS del dominio hijo publicado en `.mx` y verifica que su hash coincida con el DNSKEY que el dominio hijo publica en su propia zona. Si coincide, el segundo eslabón es válido.

3. **Resultado:** Si ambas verificaciones pasan y las firmas RRSIG están vigentes, el resolver activa el flag AD (Authenticated Data) en la respuesta, confirmando la autenticidad del dominio. Si algún eslabón falla, la cadena se rompe.

### Patrones de fallo encontrados

| Patrón | Causa | Dominios afectados |
|--------|-------|--------------------|
| DS huérfano | DS existe en zona padre pero la zona hija no responde con DNSKEY ni RRSIG | banxico.org.mx, gbm.org.mx |
| Sin DNSSEC | No hay registros criptográficos en la zona propia | tec.mx, sat.gob.mx |
| DS desalineado | DS en padre no coincide con DNSKEY en zona hija | udlap.mx |
