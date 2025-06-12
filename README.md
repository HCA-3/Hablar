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
















# crud.py

from sqlalchemy.orm import Session
from sqlalchemy import and_
from typing import List, Optional
from datetime import date
import random
import string

from models import Usuario, Mascota, Vuelo, Reserva
from schemas import UsuarioCreate, MascotaCreate, ReservaCreate
from auth import hash_password

def crear_usuario(db: Session, usuario: UsuarioCreate) -> Usuario:
    password_hash = hash_password(usuario.password)
    db_usuario = Usuario(
        nombre=usuario.nombre,
        apellido=usuario.apellido,
        email=usuario.email,
        telefono=usuario.telefono,
        password_hash=password_hash
    )
    db.add(db_usuario)
    db.commit()
    db.refresh(db_usuario)
    return db_usuario

def obtener_usuario_por_email(db: Session, email: str) -> Optional[Usuario]:
    return db.query(Usuario).filter(Usuario.email == email).first()

def obtener_usuario_por_id(db: Session, usuario_id: int) -> Optional[Usuario]:
    return db.query(Usuario).filter(Usuario.id == usuario_id).first()

def crear_mascota(db: Session, mascota: MascotaCreate, propietario_id: int) -> Mascota:
    db_mascota = Mascota(
        nombre=mascota.nombre,
        especie=mascota.especie,
        raza=mascota.raza,
        edad=mascota.edad,
        peso=mascota.peso,
        observaciones=mascota.observaciones,
        propietario_id=propietario_id
    )
    db.add(db_mascota)
    db.commit()
    db.refresh(db_mascota)
    return db_mascota

def obtener_mascotas_usuario(db: Session, usuario_id: int) -> List[Mascota]:
    return db.query(Mascota).filter(
        and_(Mascota.propietario_id == usuario_id, Mascota.activo == True)
    ).all()

def obtener_mascota_por_id(db: Session, mascota_id: int) -> Optional[Mascota]:
    return db.query(Mascota).filter(Mascota.id == mascota_id).first()
    
def buscar_vuelos(db: Session, origen: str, destino: str, fecha: date) -> List[Vuelo]:
    return db.query(Vuelo).filter(
        and_(
            Vuelo.origen.ilike(f"%{origen}%"),
            Vuelo.destino.ilike(f"%{destino}%"),
            Vuelo.fecha_salida == fecha,
            Vuelo.disponible == True
        )
    ).all()

def obtener_vuelo_por_id(db: Session, vuelo_id: int) -> Optional[Vuelo]:
    return db.query(Vuelo).filter(Vuelo.id == vuelo_id).first()

def verificar_disponibilidad_vuelo(db: Session, vuelo_id: int) -> bool:
    vuelo = obtener_vuelo_por_id(db, vuelo_id)
    if not vuelo or not vuelo.disponible:
        return False
    
    # Contar reservas activas para este vuelo
    reservas_activas = db.query(Reserva).filter(
        and_(
            Reserva.vuelo_id == vuelo_id,
            Reserva.estado == "confirmada"
        )
    ).count()
    
    return reservas_activas < vuelo.capacidad_mascotas

def generar_codigo_reserva() -> str:
    """Genera un código de reserva único de 8 caracteres"""
    return ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))

def crear_reserva(db: Session, reserva: ReservaCreate, usuario_id: int) -> Reserva:
    # Verificar que el vuelo existe y está disponible
    vuelo = obtener_vuelo_por_id(db, reserva.vuelo_id)
    if not vuelo:
        raise ValueError("Vuelo no encontrado")
    
    if not verificar_disponibilidad_vuelo(db, reserva.vuelo_id):
        raise ValueError("Vuelo no disponible")
    
    # Verificar que la mascota pertenece al usuario
    mascota = obtener_mascota_por_id(db, reserva.mascota_id)
    if not mascota or mascota.propietario_id != usuario_id:
        raise ValueError("Mascota no encontrada o no pertenece al usuario")
    
    # Calcular precio total
    precio_total = vuelo.precio_base + vuelo.precio_mascota
    
    # Generar código de reserva único
    codigo_reserva = generar_codigo_reserva()
    while db.query(Reserva).filter(Reserva.codigo_reserva == codigo_reserva).first():
        codigo_reserva = generar_codigo_reserva()
    
    db_reserva = Reserva(
        codigo_reserva=codigo_reserva,
        usuario_id=usuario_id,
        vuelo_id=reserva.vuelo_id,
        mascota_id=reserva.mascota_id,
        precio_total=precio_total,
        observaciones=reserva.observaciones
    )
    
    db.add(db_reserva)
    db.commit()
    db.refresh(db_reserva)
    return db_reserva

def obtener_reservas_usuario(db: Session, usuario_id: int) -> List[Reserva]:
    return db.query(Reserva).filter(Reserva.usuario_id == usuario_id).all()

def obtener_reserva_por_codigo(db: Session, codigo_reserva: str) -> Optional[Reserva]:
    return db.query(Reserva).filter(Reserva.codigo_reserva == codigo_reserva).first()

def verificar_password(password: str, password_hash: str) -> bool:
    from auth import verify_password
    return verify_password(password, password_hash)


















# auth.py

from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
import os

SECRET_KEY = os.getenv("SECRET_KEY", "tu-clave-secreta-muy-segura-cambiala-en-produccion")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    """Hashea una contraseña"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verifica una contraseña contra su hash"""
    return pwd_context.verify(plain_password, hashed_password)

def crear_token_acceso(email: str, expires_delta: timedelta = None) -> str:
    """Crea un token JWT de acceso"""
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode = {"sub": email, "exp": expire}
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verificar_token(token: str) -> str:
    """Verifica un token JWT y retorna el email del usuario"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None:
            return None
        return email
    except JWTError:
        return None
















# database.py

from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os


DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./aeromascotas.db") # URL de la base de datos (SQLite para desarrollo, PostgreSQL para producción)


if DATABASE_URL.startswith("postgres://"):# Para PostgreSQL en producción (Render)
    DATABASE_URL = DATABASE_URL.replace("postgres://", "postgresql://", 1)

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False} if "sqlite" in DATABASE_URL else {}
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


















# models.py

from sqlalchemy import Column, Integer, String, DateTime, Date, Float, Boolean, ForeignKey, Text
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from database import Base

class Usuario(Base):
    __tablename__ = "usuarios"
    
    id = Column(Integer, primary_key=True, index=True)
    nombre = Column(String(100), nullable=False)
    apellido = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, index=True, nullable=False)
    telefono = Column(String(20), nullable=False)
    password_hash = Column(String(255), nullable=False)
    fecha_registro = Column(DateTime(timezone=True), server_default=func.now())
    activo = Column(Boolean, default=True)
    
    # Relaciones
    mascotas = relationship("Mascota", back_populates="propietario")
    reservas = relationship("Reserva", back_populates="usuario")

class Mascota(Base):
    __tablename__ = "mascotas"
    
    id = Column(Integer, primary_key=True, index=True)
    nombre = Column(String(100), nullable=False)
    especie = Column(String(50), nullable=False)  # perro, gato, etc.
    raza = Column(String(100))
    edad = Column(Integer)
    peso = Column(Float, nullable=False)  # en kg
    observaciones = Column(Text)
    propietario_id = Column(Integer, ForeignKey("usuarios.id"), nullable=False)
    fecha_registro = Column(DateTime(timezone=True), server_default=func.now())
    activo = Column(Boolean, default=True)
    
    # Relaciones
    propietario = relationship("Usuario", back_populates="mascotas")
    reservas = relationship("Reserva", back_populates="mascota")

class Vuelo(Base):
    __tablename__ = "vuelos"
    
    id = Column(Integer, primary_key=True, index=True)
    numero_vuelo = Column(String(10), unique=True, nullable=False)
    origen = Column(String(100), nullable=False)
    destino = Column(String(100), nullable=False)
    fecha_salida = Column(Date, nullable=False)
    hora_salida = Column(String(5), nullable=False)  # HH:MM
    fecha_llegada = Column(Date, nullable=False)
    hora_llegada = Column(String(5), nullable=False)  # HH:MM
    precio_base = Column(Float, nullable=False)
    precio_mascota = Column(Float, nullable=False)
    capacidad_total = Column(Integer, nullable=False)
    capacidad_mascotas = Column(Integer, nullable=False)
    disponible = Column(Boolean, default=True)
    
    # Relaciones
    reservas = relationship("Reserva", back_populates="vuelo")

class Reserva(Base):
    __tablename__ = "reservas"
    
    id = Column(Integer, primary_key=True, index=True)
    codigo_reserva = Column(String(10), unique=True, nullable=False)
    usuario_id = Column(Integer, ForeignKey("usuarios.id"), nullable=False)
    vuelo_id = Column(Integer, ForeignKey("vuelos.id"), nullable=False)
    mascota_id = Column(Integer, ForeignKey("mascotas.id"), nullable=False)
    precio_total = Column(Float, nullable=False)
    estado = Column(String(20), default="confirmada")  # confirmada, cancelada, completada
    fecha_reserva = Column(DateTime(timezone=True), server_default=func.now())
    observaciones = Column(Text)
    
    # Relaciones
    usuario = relationship("Usuario", back_populates="reservas")
    vuelo = relationship("Vuelo", back_populates="reservas")
    mascota = relationship("Mascota", back_populates="reservas")















  # requirements.txt

fastapi==0.104.1
uvicorn[standard]==0.24.0
sqlalchemy==2.0.23
psycopg2-binary==2.9.9
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
email-validator==2.1.0
pydantic[email]==2.5.0














# schemas.py

from pydantic import BaseModel, EmailStr
from typing import Optional, List
from datetime import datetime, date

class UsuarioBase(BaseModel):
    nombre: str
    apellido: str
    email: EmailStr
    telefono: str

class UsuarioCreate(UsuarioBase):
    password: str

class UsuarioLogin(BaseModel):
    email: EmailStr
    password: str

class UsuarioResponse(UsuarioBase):
    id: int
    fecha_registro: datetime
    activo: bool
    
    class Config:
        from_attributes = True

class MascotaBase(BaseModel):
    nombre: str
    especie: str
    raza: Optional[str] = None
    edad: Optional[int] = None
    peso: float
    observaciones: Optional[str] = None

class MascotaCreate(MascotaBase):
    pass

class MascotaResponse(MascotaBase):
    id: int
    propietario_id: int
    fecha_registro: datetime
    activo: bool
    
    class Config:
        from_attributes = True

class VueloBase(BaseModel):
    numero_vuelo: str
    origen: str
    destino: str
    fecha_salida: date
    hora_salida: str
    fecha_llegada: date
    hora_llegada: str
    precio_base: float
    precio_mascota: float

class VueloResponse(VueloBase):
    id: int
    capacidad_total: int
    capacidad_mascotas: int
    disponible: bool
    
    class Config:
        from_attributes = True

class VueloSearch(BaseModel):
    origen: str
    destino: str
    fecha: date

class ReservaBase(BaseModel):
    vuelo_id: int
    mascota_id: int
    observaciones: Optional[str] = None

class ReservaCreate(ReservaBase):
    pass

class ReservaResponse(ReservaBase):
    id: int
    codigo_reserva: str
    usuario_id: int
    precio_total: float
    estado: str
    fecha_reserva: datetime
    vuelo: VueloResponse
    mascota: MascotaResponse
    
    class Config:
        from_attributes = True
















# seed_data.py


from sqlalchemy.orm import Session
from database import SessionLocal, engine
from models import Base, Vuelo
from datetime import date, timedelta

Base.metadata.create_all(bind=engine)

def crear_vuelos_ejemplo():
    """Crea algunos vuelos de ejemplo para probar la aplicación"""
    db = SessionLocal()
    
    if db.query(Vuelo).first():
        print("Ya existen vuelos en la base de datos")
        db.close()
        return
    
    vuelos_ejemplo = [
        {
            "numero_vuelo": "AM001",
            "origen": "Madrid",
            "destino": "Barcelona",
            "fecha_salida": date.today() + timedelta(days=7),
            "hora_salida": "08:00",
            "fecha_llegada": date.today() + timedelta(days=7),
            "hora_llegada": "09:30",
            "precio_base": 150.0,
            "precio_mascota": 50.0,
            "capacidad_total": 180,
            "capacidad_mascotas": 20
        },
        {
            "numero_vuelo": "AM002",
            "origen": "Barcelona",
            "destino": "Madrid",
            "fecha_salida": date.today() + timedelta(days=7),
            "hora_salida": "14:00",
            "fecha_llegada": date.today() + timedelta(days=7),
            "hora_llegada": "15:30",
            "precio_base": 150.0,
            "precio_mascota": 50.0,
            "capacidad_total": 180,
            "capacidad_mascotas": 20
        },
        {
            "numero_vuelo": "AM003",
            "origen": "Madrid",
            "destino": "Sevilla",
            "fecha_salida": date.today() + timedelta(days=8),
            "hora_salida": "10:00",
            "fecha_llegada": date.today() + timedelta(days=8),
            "hora_llegada": "11:15",
            "precio_base": 120.0,
            "precio_mascota": 40.0,
            "capacidad_total": 150,
            "capacidad_mascotas": 15
        },
        {
            "numero_vuelo": "AM004",
            "origen": "Valencia",
            "destino": "Bilbao",
            "fecha_salida": date.today() + timedelta(days=9),
            "hora_salida": "16:00",
            "fecha_llegada": date.today() + timedelta(days=9),
            "hora_llegada": "17:45",
            "precio_base": 180.0,
            "precio_mascota": 60.0,
            "capacidad_total": 200,
            "capacidad_mascotas": 25
        },
        {
            "numero_vuelo": "AM005",
            "origen": "Málaga",
            "destino": "Madrid",
            "fecha_salida": date.today() + timedelta(days=10),
            "hora_salida": "12:00",
            "fecha_llegada": date.today() + timedelta(days=10),
            "hora_llegada": "13:30",
            "precio_base": 140.0,
            "precio_mascota": 45.0,
            "capacidad_total": 160,
            "capacidad_mascotas": 18
        }
    ]
    
    for vuelo_data in vuelos_ejemplo:
        vuelo = Vuelo(**vuelo_data)
        db.add(vuelo)
    
    db.commit()
    print(f"Se crearon {len(vuelos_ejemplo)} vuelos de ejemplo")
    db.close()

if __name__ == "__main__":
    crear_vuelos_ejemplo()













# main.py


from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.orm import Session
from typing import List, Optional
import uvicorn
from datetime import datetime, date
from database import get_db, engine
from models import Base, Usuario, Mascota, Vuelo, Reserva
from schemas import (
    UsuarioCreate, UsuarioResponse, UsuarioLogin,
    MascotaCreate, MascotaResponse,
    VueloResponse, VueloSearch,
    ReservaCreate, ReservaResponse
)
from crud import (
    crear_usuario, obtener_usuario_por_email, verificar_password,
    crear_mascota, obtener_mascotas_usuario,
    buscar_vuelos, crear_reserva, obtener_reservas_usuario
)
from auth import crear_token_acceso, verificar_token

Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="AeroMascotas API",
    description="API para empresa de vuelos con mascotas",
    version="1.0.0"
)

security = HTTPBearer()

async def obtener_usuario_actual(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
):
    token = credentials.credentials
    email = verificar_token(token)
    if not email:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token inválido"
        )
    
    usuario = obtener_usuario_por_email(db, email)
    if not usuario:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Usuario no encontrado"
        )
    return usuario

@app.get("/")
async def root():
    return {"mensaje": "Bienvenido a AeroMascotas API"}

@app.post("/usuarios/registro", response_model=UsuarioResponse)
async def registrar_usuario(usuario: UsuarioCreate, db: Session = Depends(get_db)):
    # Verificar si el usuario ya existe
    usuario_existente = obtener_usuario_por_email(db, usuario.email)
    if usuario_existente:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="El email ya está registrado"
        )
    
    nuevo_usuario = crear_usuario(db, usuario)
    return nuevo_usuario

@app.post("/usuarios/login")
async def login_usuario(credenciales: UsuarioLogin, db: Session = Depends(get_db)):
    usuario = obtener_usuario_por_email(db, credenciales.email)
    if not usuario or not verificar_password(credenciales.password, usuario.password_hash):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas"
        )
    
    token = crear_token_acceso(usuario.email)
    return {"access_token": token, "token_type": "bearer"}

@app.post("/mascotas", response_model=MascotaResponse)
async def registrar_mascota(
    mascota: MascotaCreate,
    usuario_actual: Usuario = Depends(obtener_usuario_actual),
    db: Session = Depends(get_db)
):
    nueva_mascota = crear_mascota(db, mascota, usuario_actual.id)
    return nueva_mascota

@app.get("/mascotas", response_model=List[MascotaResponse])
async def obtener_mis_mascotas(
    usuario_actual: Usuario = Depends(obtener_usuario_actual),
    db: Session = Depends(get_db)
):
    mascotas = obtener_mascotas_usuario(db, usuario_actual.id)
    return mascotas

@app.get("/vuelos/buscar", response_model=List[VueloResponse])
async def buscar_vuelos_disponibles(
    origen: str,
    destino: str,
    fecha: date,
    db: Session = Depends(get_db)
):
    vuelos = buscar_vuelos(db, origen, destino, fecha)
    return vuelos

@app.post("/reservas", response_model=ReservaResponse)
async def crear_reserva_vuelo(
    reserva: ReservaCreate,
    usuario_actual: Usuario = Depends(obtener_usuario_actual),
    db: Session = Depends(get_db)
):
    nueva_reserva = crear_reserva(db, reserva, usuario_actual.id)
    return nueva_reserva

@app.get("/reservas", response_model=List[ReservaResponse])
async def obtener_mis_reservas(
    usuario_actual: Usuario = Depends(obtener_usuario_actual),
    db: Session = Depends(get_db)
):
    reservas = obtener_reservas_usuario(db, usuario_actual.id)
    return reservas

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)







