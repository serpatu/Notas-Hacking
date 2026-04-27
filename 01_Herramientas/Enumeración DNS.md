# Enumeración DNS

**Categoría:** Reconocimiento
**Etiquetas:** #dns #recon #network

## Herramientas Clave

### 1. Dig (Domain Information Groper)
El estándar en Linux para consultas DNS.

- **Registro A (IP):** `dig dominio.com`
- **Registro MX (Correos):** `dig dominio.com mx`
- **Inversa (IP a Dominio):** `dig -x <IP>`
- **Transferencia de Zona (¡Importante en HTB!):** `dig axfr @<IP_DNS> dominio.com`

### 2. Nslookup / Host
Alternativas más simples.

- `nslookup dominio.com`
- `host <IP>` (Búsqueda inversa rápida)

### 3. DNSRecon
Herramienta automatizada para enumeración más profunda.

- `dnsrecon -d dominio.com`
