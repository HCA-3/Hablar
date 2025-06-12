# readmi

# AeroMascotas API

API REST para una empresa de vuelos con mascotas desarrollada con FastAPI.

## Características

- ✅ Registro y autenticación de usuarios
- ✅ Gestión de mascotas (CRUD)
- ✅ Búsqueda de vuelos por origen, destino y fecha
- ✅ Sistema de reservas con verificación de disponibilidad
- ✅ Base de datos con SQLAlchemy
- ✅ Autenticación JWT
- ✅ Documentación automática con Swagger
- ✅ Preparado para despliegue en Render

## Instalación

1. Clona el repositorio
2. Instala las dependencias:
```bash
pip install -r requirements.txt
```

3. Crea datos de ejemplo (opcional):
```bash
python seed_data.py
```

4. Ejecuta la aplicación:
```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

## Despliegue en Render

El comando de inicio para Render es:
```bash
uvicorn main:app --host 0.0.0.0 --port $PORT
```

Asegúrate de configurar las siguientes variables de entorno en Render:
- `DATABASE_URL`: URL de tu base de datos PostgreSQL
- `SECRET_KEY`: Clave secreta para JWT (genera una segura)

## Endpoints Principales

### Usuarios
- `POST /usuarios/registro` - Registrar nuevo usuario
- `POST /usuarios/login` - Iniciar sesión

### Mascotas
- `POST /mascotas` - Registrar mascota
- `GET /mascotas` - Obtener mis mascotas

### Vuelos
- `GET /vuelos/buscar?origen=Madrid&destino=Barcelona&fecha=2024-01-15` - Buscar vuelos

### Reservas
- `POST /reservas` - Crear reserva
- `GET /reservas` - Obtener mis reservas

## Documentación

Una vez ejecutada la aplicación, puedes acceder a:
- Documentación Swagger: `http://localhost:8000/docs`
- Documentación ReDoc: `http://localhost:8000/redoc`

## Modelos de Datos

### Usuario
- Información personal (nombre, apellido, email, teléfono)
- Autenticación segura con hash de contraseña
- Relación con mascotas y reservas

### Mascota
- Información de la mascota (nombre, especie, raza, edad, peso)
- Vinculada a un propietario
- Observaciones especiales

### Vuelo
- Información del vuelo (origen, destino, fechas, horarios)
- Precios diferenciados (base + mascota)
- Control de capacidad

### Reserva
- Código único de reserva
- Vincula usuario, vuelo y mascota
- Control de estado y precio total

## Ejemplo de Uso

1. Registrar usuario:
```json
POST /usuarios/registro
{
  "nombre": "Juan",
  "apellido": "Pérez",
  "email": "juan@email.com",
  "telefono": "+34123456789",
  "password": "mipassword123"
}
```

2. Iniciar sesión:
```json
POST /usuarios/login
{
  "email": "juan@email.com",
  "password": "mipassword123"
}
```

3. Registrar mascota:
```json
POST /mascotas
{
  "nombre": "Max",
  "especie": "perro",
  "raza": "Golden Retriever",
  "edad": 3,
  "peso": 25.5,
  "observaciones": "Muy tranquilo durante los viajes"
}
```

4. Buscar vuelos:
```
GET /vuelos/buscar?origen=Madrid&destino=Barcelona&fecha=2024-01-15
```

5. Crear reserva:
```json
POST /reservas
{
  "vuelo_id": 1,
  "mascota_id": 1,
  "observaciones": "Primera vez que viaja"
}
```
