# main. py

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

# Endpoints de Usuarios
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

# Endpoints de Mascotas
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
