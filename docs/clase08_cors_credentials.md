# Clase 8 - Tarea 2: CORS_ALLOWED_ORIGINS vs. CORS_ALLOW_CREDENTIALS

## CORS_ALLOWED_ORIGINS

Es la lista de orígenes (protocolo + dominio + puerto) a los que el servidor les permite
leer sus respuestas desde el navegador. Si el `Origin` de la petición está en la lista,
`django-cors-headers` agrega el header `Access-Control-Allow-Origin` con ese origen; si no
está, no lo agrega y el navegador le bloquea la respuesta al JavaScript de esa página.

En TaskFlow:

```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "https://taskflow-app.com",
]
```

## CORS_ALLOW_CREDENTIALS

Por defecto, en una petición entre orígenes el navegador **no** manda cookies ni
credenciales HTTP, y aunque las mande, no deja que el JavaScript lea la respuesta.
Con `CORS_ALLOW_CREDENTIALS = True` el servidor responde además con
`Access-Control-Allow-Credentials: true`, que habilita las dos cosas. Del lado del
frontend también hay que pedirlo explícitamente: `fetch(url, { credentials: "include" })`.

## ¿En qué escenario se necesita?

Cuando el frontend está en **otro origen** y se autentica con **cookies**, por ejemplo con
la sesión de Django (`SessionAuthentication`) o con el JWT guardado en una cookie
`HttpOnly`. Sin `CORS_ALLOW_CREDENTIALS` la cookie de sesión no viajaría y el usuario
quedaría como anónimo.

En nuestro caso **no hace falta**: la API se usa con JWT en el header
`Authorization: Bearer ...`, que lo agrega el propio código del frontend. Ese header ya
está permitido por la configuración de CORS (`access-control-allow-headers` incluye
`authorization`) y no cuenta como "credencial" en el sentido de CORS.

## Precauciones si se activa

- Con credenciales, `Access-Control-Allow-Origin` no puede ser `*`: tiene que ser un
  origen concreto. Por eso nunca hay que combinarlo con `CORS_ALLOW_ALL_ORIGINS = True`,
  porque cualquier sitio podría hacer peticiones con la sesión del usuario y leer la respuesta.
- Al usar cookies vuelve a importar **CSRF**: hay que agregar el origen del frontend en
  `CSRF_TRUSTED_ORIGINS` y mandar el token CSRF en las peticiones que modifican datos.
