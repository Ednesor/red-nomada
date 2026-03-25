# Flujo de Autenticación: Red-Nómada MVP

## Estrategia: JWT con Refresh Token

---

## Flujo de Registro

1. Usuario envía `POST /auth/registro` con `{ email, nombre, password }`
2. Backend valida campos (Zod), verifica email no duplicado
3. Hashea password con **bcrypt** (salt rounds: 12)
4. Crea registro en tabla `Usuario` con rol `EXPLORADOR`
5. Genera `accessToken` (JWT, expira en **15 minutos**) y `refreshToken` (expira en **7 días**)
6. Retorna tokens + datos básicos del usuario

---

## Flujo de Login

1. Usuario envía `POST /auth/login` con `{ email, password }`
2. Backend busca usuario por email
3. Compara password con hash almacenado (bcrypt)
4. Si es válido → genera par de tokens nuevos
5. Si es inválido → 401 sin revelar si el email existe o no

---

## Refresh Token

- **Access Token:** JWT firmado con secreto, payload: `{ userId, rol, iat, exp }`
- **Refresh Token:** string opaco (UUID v4), almacenado hasheado en BD
- Cuando el access token expira, el frontend usa `POST /auth/refresh`
- Se implementa **rotación de refresh tokens**: cada uso genera un nuevo par
- Si un refresh token se usa dos veces → se invalidan TODOS los tokens del usuario (posible robo)

---

## Flujo en Frontend (PWA)

```
1. Login/Registro → guardar tokens en memoria (Zustand store)
2. Cada request → interceptor Axios agrega header Authorization
3. Si 401 → intentar refresh automático
4. Si refresh falla → redirigir a /login, limpiar store
5. Al cerrar app → tokens persisten en sessionStorage (no localStorage por seguridad)
```

---

## Seguridad

- **No** almacenar tokens en `localStorage` (vulnerable a XSS)
- Usar `sessionStorage` o bien `httpOnly cookies` en producción
- Rate limit en `/auth/login`: **5 intentos** por IP en 15 minutos
- Rate limit en `/auth/registro`: **3 registros** por IP en 1 hora
- Passwords: mínimo 8 caracteres, al menos 1 mayúscula y 1 número
