# Flujo de Autenticación: Red-Nómada MVP

## Estrategia: JWT con Refresh Token (Spring Security)

---

## Flujo de Registro

1. Usuario envía `POST /auth/registro` con `{ email, nombre, password }`
2. Backend valida campos (`@Valid` + Bean Validation), verifica email no duplicado
3. Hashea password con **BCryptPasswordEncoder** (Spring Security)
4. Crea registro en tabla `Usuario` con rol `EXPLORADOR`
5. Genera `accessToken` (JWT firmado con `jjwt`, expira en **15 minutos**) y `refreshToken` (expira en **7 días**)
6. Retorna tokens + datos básicos del usuario

---

## Flujo de Login

1. Usuario envía `POST /auth/login` con `{ email, password }`
2. Backend busca usuario por email (Spring Data JPA)
3. Compara password con hash almacenado (`BCryptPasswordEncoder.matches()`)
4. Si es válido → genera par de tokens nuevos
5. Si es inválido → 401 sin revelar si el email existe o no

---

## Refresh Token

- **Access Token:** JWT firmado con secreto (HMAC-SHA256 vía `jjwt`), payload: `{ userId, rol, iat, exp }`
- **Refresh Token:** string opaco (UUID v4), almacenado hasheado en BD
- Cuando el access token expira, el frontend usa `POST /auth/refresh`
- Se implementa **rotación de refresh tokens**: cada uso genera un nuevo par
- Si un refresh token se usa dos veces → se invalidan TODOS los tokens del usuario (posible robo)

---

## Spring Security Filter Chain

```
SecurityFilterChain:
1. JwtAuthenticationFilter (OncePerRequestFilter)
   → Extrae token del header Authorization
   → Valida firma y expiración con jjwt
   → Setea SecurityContextHolder con UserDetails
2. Endpoints públicos: /auth/**, /api/v1/lugares (GET)
3. Endpoints protegidos: todo lo demás requiere Bearer token válido
4. Roles: EXPLORADOR (default), ADMIN (gestión)
```

---

## Flujo en Frontend (React Native)

```
1. Login/Registro → guardar tokens en expo-secure-store (encriptado nativo)
2. Cada request → interceptor Axios agrega header Authorization
3. Si 401 → intentar refresh automático
4. Si refresh falla → redirigir a pantalla de Login, limpiar stores
5. Al cerrar app → tokens persisten en secure-store (encriptado por el OS)
```

---

## Seguridad

- Almacenar tokens en `expo-secure-store` (Keychain iOS / Keystore Android)
- **No** usar AsyncStorage para tokens (no encriptado)
- Rate limit en `/auth/login`: **5 intentos** por IP en 15 minutos (Bucket4j)
- Rate limit en `/auth/registro`: **3 registros** por IP en 1 hora
- Passwords: mínimo 8 caracteres, al menos 1 mayúscula y 1 número
- CORS configurado en Spring Security para permitir solo orígenes autorizados
