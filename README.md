from pydantic import BaseModel, EmailStr
from datetime import datetime
from typing import Optional, List

class UserBase(BaseModel):
    nombre: str
    email: EmailStr
    telefono: Optional[str] = None

class UserCreate(UserBase):
    pass

class UserUpdate(BaseModel):
    nombre: Optional[str] = None
    email: Optional[EmailStr] = None
    telefono: Optional[str] = None

class User(UserBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True

class PetBase(BaseModel):
    nombre_mascota: str
    especie: str
    raza: Optional[str] = None
    peso: float
    edad: Optional[int] = None
    observaciones: Optional[str] = None

class PetCreate(PetBase):
    id_usuario: int

class PetUpdate(BaseModel):
    nombre_mascota: Optional[str] = None
    especie: Optional[str] = None
    raza: Optional[str] = None
    peso: Optional[float] = None
    edad: Optional[int] = None
    observaciones: Optional[str] = None

class Pet(PetBase):
    id: int
    id_usuario: int
    created_at: datetime
    
    class Config:
        from_attributes = True

class FlightBase(BaseModel):
    numero_vuelo: str
    origen: str
    destino: str
    fecha_salida: datetime
    fecha_llegada: datetime
    precio_base: float
    precio_mascota: float
    capacidad_total: int
    capacidad_mascotas: int
    disponible: bool = True

class FlightCreate(FlightBase):
    pass

class FlightUpdate(BaseModel):
    numero_vuelo: Optional[str] = None
    origen: Optional[str] = None
    destino: Optional[str] = None
    fecha_salida: Optional[datetime] = None
    fecha_llegada: Optional[datetime] = None
    precio_base: Optional[float] = None
    precio_mascota: Optional[float] = None
    capacidad_total: Optional[int] = None
    capacidad_mascotas: Optional[int] = None
    disponible: Optional[bool] = None

class Flight(FlightBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True

class FlightSearch(BaseModel):
    origen: str
    destino: str
    fecha: datetime

class ReservationBase(BaseModel):
    id_usuario: int
    id_vuelo: int
    id_mascota: int

class ReservationCreate(ReservationBase):
    pass

class ReservationUpdate(BaseModel):
    estado: Optional[str] = None

class Reservation(ReservationBase):
    id: int
    codigo_reserva: str
    estado: str
    precio_total: float
    fecha_reserva: datetime
    
    class Config:
        from_attributes = True

class PurchaseBase(BaseModel):
    metodo_pago: str
    referencia_pago: Optional[str] = None

class PurchaseCreate(PurchaseBase):
    id_reserva: int

class PurchaseUpdate(BaseModel):
    estado_pago: Optional[str] = None
    referencia_pago: Optional[str] = None

class Purchase(PurchaseBase):
    id: int
    id_reserva: int
    estado_pago: str
    monto: float
    fecha_compra: datetime
    
    class Config:
        from_attributes = True
        #11
