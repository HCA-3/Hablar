from sqlalchemy.orm import Session
from sqlalchemy import and_, func
from datetime import datetime, date
import models
import schemas
import random
import string

def get_user(db: Session, user_id: int):
    return db.query(models.User).filter(models.User.id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(models.User).filter(models.User.email == email).first()

def get_users(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.User).offset(skip).limit(limit).all()

def create_user(db: Session, user: schemas.UserCreate):
    db_user = models.User(**user.dict())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

def update_user(db: Session, user_id: int, user: schemas.UserUpdate):
    db_user = db.query(models.User).filter(models.User.id == user_id).first()
    if db_user:
        update_data = user.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(db_user, field, value)
        db.commit()
        db.refresh(db_user)
    return db_user

def delete_user(db: Session, user_id: int):
    db_user = db.query(models.User).filter(models.User.id == user_id).first()
    if db_user:
        db.delete(db_user)
        db.commit()
    return db_user


def get_pet(db: Session, pet_id: int):
    return db.query(models.Pet).filter(models.Pet.id == pet_id).first()

def get_pets_by_user(db: Session, user_id: int):
    return db.query(models.Pet).filter(models.Pet.id_usuario == user_id).all()

def get_pets(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.Pet).offset(skip).limit(limit).all()

def create_pet(db: Session, pet: schemas.PetCreate):
    db_pet = models.Pet(**pet.dict())
    db.add(db_pet)
    db.commit()
    db.refresh(db_pet)
    return db_pet

def update_pet(db: Session, pet_id: int, pet: schemas.PetUpdate):
    db_pet = db.query(models.Pet).filter(models.Pet.id == pet_id).first()
    if db_pet:
        update_data = pet.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(db_pet, field, value)
        db.commit()
        db.refresh(db_pet)
    return db_pet

def delete_pet(db: Session, pet_id: int):
    db_pet = db.query(models.Pet).filter(models.Pet.id == pet_id).first()
    if db_pet:
        db.delete(db_pet)
        db.commit()
    return db_pet

def get_flight(db: Session, flight_id: int):
    return db.query(models.Flight).filter(models.Flight.id == flight_id).first()

def get_flights(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.Flight).offset(skip).limit(limit).all()

def search_flights(db: Session, origen: str, destino: str, fecha: date):
    return db.query(models.Flight).filter(
        and_(
            models.Flight.origen.ilike(f"%{origen}%"),
            models.Flight.destino.ilike(f"%{destino}%"),
            func.date(models.Flight.fecha_salida) == fecha,
            models.Flight.disponible == True
        )
    ).all()

def create_flight(db: Session, flight: schemas.FlightCreate):
    db_flight = models.Flight(**flight.dict())
    db.add(db_flight)
    db.commit()
    db.refresh(db_flight)
    return db_flight

def update_flight(db: Session, flight_id: int, flight: schemas.FlightUpdate):
    db_flight = db.query(models.Flight).filter(models.Flight.id == flight_id).first()
    if db_flight:
        update_data = flight.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(db_flight, field, value)
        db.commit()
        db.refresh(db_flight)
    return db_flight

def delete_flight(db: Session, flight_id: int):
    db_flight = db.query(models.Flight).filter(models.Flight.id == flight_id).first()
    if db_flight:
        db.delete(db_flight)
        db.commit()
    return db_flight

def check_flight_availability(db: Session, flight_id: int):
    flight = db.query(models.Flight).filter(models.Flight.id == flight_id).first()
    if not flight or not flight.disponible:
        return False

    reserved_pets = db.query(models.Reservation).filter(
        and_(
            models.Reservation.id_vuelo == flight_id,
            models.Reservation.estado.in_(["pendiente", "confirmada"])
        )
    ).count()
    
    return reserved_pets < flight.capacidad_mascotas

def generate_reservation_code():
    return ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))

def get_reservation(db: Session, reservation_id: int):
    return db.query(models.Reservation).filter(models.Reservation.id == reservation_id).first()

def get_reservations_by_user(db: Session, user_id: int):
    return db.query(models.Reservation).filter(models.Reservation.id_usuario == user_id).all()

def get_reservations(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.Reservation).offset(skip).limit(limit).all()

def create_reservation(db: Session, reservation: schemas.ReservationCreate):

    if not check_flight_availability(db, reservation.id_vuelo):
        return None

    flight = get_flight(db, reservation.id_vuelo)
    if not flight:
        return None
    
    total_price = flight.precio_base + flight.precio_mascota
    
    db_reservation = models.Reservation(
        **reservation.dict(),
        codigo_reserva=generate_reservation_code(),
        precio_total=total_price
    )
    db.add(db_reservation)
    db.commit()
    db.refresh(db_reservation)
    return db_reservation

def update_reservation(db: Session, reservation_id: int, reservation: schemas.ReservationUpdate):
    db_reservation = db.query(models.Reservation).filter(models.Reservation.id == reservation_id).first()
    if db_reservation:
        update_data = reservation.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(db_reservation, field, value)
        db.commit()
        db.refresh(db_reservation)
    return db_reservation

def delete_reservation(db: Session, reservation_id: int):
    db_reservation = db.query(models.Reservation).filter(models.Reservation.id == reservation_id).first()
    if db_reservation:
        db.delete(db_reservation)
        db.commit()
    return db_reservation

def get_purchase(db: Session, purchase_id: int):
    return db.query(models.Purchase).filter(models.Purchase.id == purchase_id).first()

def get_purchase_by_reservation(db: Session, reservation_id: int):
    return db.query(models.Purchase).filter(models.Purchase.id_reserva == reservation_id).first()

def get_purchases(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.Purchase).offset(skip).limit(limit).all()

def create_purchase(db: Session, purchase: schemas.PurchaseCreate):
    # Get reservation to calculate amount
    reservation = get_reservation(db, purchase.id_reserva)
    if not reservation:
        return None
    
    db_purchase = models.Purchase(
        **purchase.dict(),
        monto=reservation.precio_total
    )
    db.add(db_purchase)
    db.commit()
    db.refresh(db_purchase)

    update_reservation(db, purchase.id_reserva, schemas.ReservationUpdate(estado="confirmada"))
    
    return db_purchase

def update_purchase(db: Session, purchase_id: int, purchase: schemas.PurchaseUpdate):
    db_purchase = db.query(models.Purchase).filter(models.Purchase.id == purchase_id).first()
    if db_purchase:
        update_data = purchase.dict(exclude_unset=True)
        for field, value in update_data.items():
            setattr(db_purchase, field, value)
        db.commit()
        db.refresh(db_purchase)
    return db_purchase

def delete_purchase(db: Session, purchase_id: int):
    db_purchase = db.query(models.Purchase).filter(models.Purchase.id == purchase_id).first()
    if db_purchase:
        db.delete(db_purchase)
        db.commit()
    return db_purchase
