
# Máquina: Appointment

**IP:** 10.129.59.28
**OS:** Linux
**Dificultad:** Very Easy
**Técnica Clave:** [[SQL Injection]]
**Fecha:** 2025-11-21
**Skills ganadas:** #SQLi #Web #AuthBypass

---

## 1. Reconocimiento
Escaneo inicial de puertos:

```bash
nmap -p- --min-rate 5000 10.129.59.28
```

## 2. Enumeración Web
Accedemos mediante el navegador a: `http://10.129.59.28`

**Observaciones:**
- Se muestra un formulario de **Login** (Usuario y Contraseña).
- No parece haber enlaces a otras páginas ni registros.
- Probamos credenciales por defecto (`admin` / `admin`), pero el acceso es denegado.

**Vector de ataque potencial:**
Dado que es un formulario de autenticación simple, es muy probable que sea vulnerable a **SQL Injection (SQLi)** en el campo de usuario.

## 3. Explotación: Inyección "Llave Maestra" (Sin usuario conocido)

No conocemos ningún usuario válido, pero no importa. Usamos una **Inyección SQL Tautológica** para obligar a la base de datos a devolvernos el primer registro disponible (que suele ser el Administrador).

**Payload inyectado (en el campo Username):**
`' OR 1=1 #`

**Comportamiento:**
1. **`'`**: Cerramos la comilla del campo username.
2. **`OR 1=1`**: Condición siempre verdadera ("True"). Esto selecciona **todos** los usuarios de la tabla.
3. **`#`**: Comenta el resto de la consulta, haciendo que **el campo de contraseña sea totalmente ignorado**.

**Resultado:**
La aplicación nos loguea automáticamente como el primer usuario de la base de datos, dándonos acceso de administrador sin saber su nombre ni su clave.